# REST_GraphQL_article
# Django: When REST May Not Be Enough and How a GraphQL Layer Can Help Save You

## Introduction
In this article, we'll present a common problem faced by maintainers of a mature REST-based API which has been built using the Django REST Framework. We'll then proceed to demonstrate how GraphQL can provide a solution to this problem and how this solution can be applied by adding just two lines of code to your Django project via use of the GraphWrap library.
The Set-Up
P. Walnuts & Co. are a well established publishing company. They have a mature REST based API which is built using the Django REST Framework (DRF). This API, among other things, allows a consumer to fetch information about books or authors . This single API has three primary consumers: the P. Walnuts & Co website, the P. Walnuts & Co Android/OS App and external clients.
Following the typical REST paradigm, an API consumer can find details of all author's they're authorised to view via a "list" /author/ endpoint and can view details of a specific author via a "detail" /author/id/ endpoint. Similar endpoints are exposed for book . Under the hood, these endpoints are served by the following DRF serialisers:

```
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
       model = Author
       fields = ['name', 'active', 'profile']
  
class BookSerializer(serializers.ModelSerializer):
    author = serializers.HyperlinkedRelatedField(
          view_name='author-detail', read_only=True)
    class Meta:
        model = Book
        fields = ['author', 'title', 'page_count']

```

## The Problem
The company website includes a page where user's can browse the books available. When a book is clicked on, we redirect to a page which gives some more information about the book and its author. To fetch the data to serve this page, the web client calls, for example /book/1/ , which would give a response which looks like this:
```
{
  'author': '/author/1/',
  'title': 'Lovely Gardens',
  'page_count': 200,
}
```
As we can see, the web client doesn't get much information about the books author here. To get that data, they need to make a second call to /author/1/ . This is the well-known n+1 problem. It can lead to an inefficient website and overly complex front-end architecture; it's something we want to avoid! So how can we fix this problem?
Attempting a Solution in Native DRF
The obvious solution to satisfy our web client here is to simply expose more author fields on our /book endpoint. We'll add an author_full field to the BookSerializerlike so:

```
class BookSerializer(serializers.ModelSerializer):
    author = serializers.HyperlinkedRelatedField(
          view_name='author-detail', read_only=True)
    author_full = AuthorSerializer(source='author_full')
    class Meta:
        model = Book
        fields = ['author', 'author_full', 'title', 'page_count']
```

Now when a our web client requests /book/1/ , they get a full representation of the author as well:
```
{
  'author': '/author/1/',
  'title': 'Lovely Gardens',
  'page_count': 200,
  'author_full': {
      'name': 'Emilie B.',
      'active': True,
      'profile': '/profile/1/',
   }
}
```
Our web clients can now serve the "book detail" page using a single GET request. Everyone's happy!
The next day, our app client comes to us and complains that the call to /book/1/ has become slower and is returning information unnecessary for the app. Now what?
In attempting to satisfy one of our API clients, we have inadvertently introduced a whole new class of problems for our other clients. Not only that, but by exposing author fields on the /book endpoint, we're introducing unnecessary coupling in our API architecture, and we all know what kind of issues that can lead to.
It's starting to feel like that unless we start building an API per client (which of course we do not want), we're a bit stuck.

## How GraphQL Can Help Us
Enter GraphQL: GraphQL is designed so that the client decides what info it receives from the server, not the other way around. Whilst many great packages exist to create a Python GraphQL API from scratch, migrating a mature production REST API to use one of these frameworks is not so simple. Not only that, it may be that we are really in love with developing using the Django REST Framework, and don't want to switch to a new library just to solve this issue.

## Applying a GraphQL Layer via GraphWrap
GraphWrap is a python library which, by adding only two lines of code to your Django project, can extend an existing Django Rest Framework API with a GraphQL interface. This is achieved by leveraging Graphene-Django to dynamically build, at runtime, a GraphQL ObjectType for each Django REST view in your API. These ObjectTypes are then glued together to form a GraphQL schema which has the same "shape" as your existing REST API. Note that GraphWrap is not designed to build a GraphQL schema to replace your existing REST API, but rather extend it to offer an additional fully compliant GraphQL-queryable interface.
The API development team at P. Walnuts & Co. decide to add GraphWrap into their project. After installing via
```
pip install graph_wrap
```
the team expose the new /graphql endpoint on their API by adding the the graphql_view to their urlpatterns :
from rest_framework import routers

```
from graph_wrap.django_rest_framework.graphql_view import graphql_view  

urlpatterns = [
     ...,
     path(r'/graphql/', view=graphql_view), 
]
```
With this new /graphqlendpoint in place, we can now stop overexposing the author fields and instead simply expose author as a URL as we did originally:
```
class BookSerializer(serializers.ModelSerializer):
    author = serializers.HyperlinkedRelatedField(
          view_name='author-detail', read_only=True)
    class Meta:
        model = Book
        fields = ['author', 'title', 'page_count']
```
This keeps our app client, who don't care about the nested author fields happy. Our web client, who is interested in retrieving the nested author fields, can then do so via a query to the new /graphql endpoint:
```
query {
      book(id: 1) {
          title
          author {
              name
              active
          }
      }
  }
```
This will give a response which looks something like this:
```
'data': {
      'title': 'Lovely Gardens',
      'author': {
          'name': 'Emilie B.',
          'active': True,
       }
  }
```
This /graphql endpoint allows each client to decide for themselves what information they want from the book endpoint. Both our web client and our app client are now happy!
The most attractive feature of the new /graphql end point for the back-end team (apart from keeping the front-end happy of course!) is that it requires very little extra maintenance due to the following features of GraphWrap:
The dynamic nature of the build of the GraphQL layer means that you can continue to develop your existing REST based API and know that the GraphQL schema will be kept up-to-date automatically.
Since the GraphQL layer is using the REST API under-the-hood, you can be sure that important things like serialization, authentication, authorization and filtering will be consistent between your REST view and the corresponding GraphQL type.

For more on GraphWrap, see https://github.com/PaulGilmartin/graph_wrap
