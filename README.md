# elasticsearch-dsl-examples
Code examples showing how to use elasticsearch-dsl.py.

# Elasticsearch Examples in Python

This notebook shows working examples using `elasticsearch-dsl.py` to interface with `Elasticsearch`.  This workbook assumes `jupyter` is running on the same host as a single-node `Elasticsearch` instance.  The examples are geared to `python 3.6`.  They exercise an index called `bank` that was created using the sample data set described [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/_exploring_your_data.html) in the Elasticsearch getting started tutorial.

This is version 0.1 dated 20/AUG/2017 and prepared by Dude Revolucion.  Updated versions will include more examples as I continue learning how to use the high-level Python interface to Elasticsearch.


```python
from elasticsearch import Elasticsearch
from elasticsearch_dsl import Search,Q,A
from elasticsearch_dsl.query import Bool
```

## Preliminaries

Create a connection to the local `Elasticsearch` instance and use that client to create a new search request on the `bank` index.


```python
client = Elasticsearch()
s = Search(using=client,index='bank')
```

## A Query

Python  version of the following example that can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/_executing_searches.html) in the Elasticsearch getting started tutorial.  Returns all accounts containing the term "mill" and "lane" in the address.

The example below corresponds to the following DSL syntax.

`curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
'`


```python
q = Bool(
         must = [Q('match',address='mill'),Q('match',address='lane')]
        )
s.query = q
resp = s.execute()
for h in resp :
    print(h.to_dict())
```

    {'account_number': 136, 'balance': 45801, 'firstname': 'Winnie', 'lastname': 'Holland', 'age': 38, 'gender': 'M', 'address': '198 Mill Lane', 'employer': 'Neteria', 'email': 'winnieholland@neteria.com', 'city': 'Urie', 'state': 'IL'}


## A Query with Filter

Python version of the following example that can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/_executing_filters.html) in the Elasticsearch getting started tutorial.  This example uses a bool query to return all accounts with balances between 20000 and 30000, inclusive. In other words, we want to find accounts with a balance that is greater than or equal to 20000 and less than or equal to 30000.

The example below corresponds to the following DSL syntax.

`curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
'`


```python
q = Bool( \
         must = [Q('match_all')],
         filter = [Q('range',balance={'gte':20000,'lte':30000})]
)
s.query = q
resp = s.execute(ignore_cache=True)
for h in resp :
    print(h.account_number,h.balance)
```

    253 20240
    400 20685
    520 27987
    645 29362
    734 20325
    784 25291
    880 22575
    14 20480
    19 27894
    204 27714


## An Aggregation

Python version of the following example that can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/_executing_aggregations.html) in the Elasticsearch getting started tutorial.  This example groups all the accounts by state, and then returns the top 10 (default) states sorted by count descending (also default):

The example below corresponds to the following DSL syntax.

`curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
'`


```python
q = Bool( \
         must = [Q('match_all')],
)
s.query = q
a = A('terms',field='state.keyword')
s.aggs.bucket('group_by_state',a)
resp = s.execute(ignore_cache=True)
print( resp.aggregations['group_by_state'].buckets )
```

    [{'key': 'ID', 'doc_count': 27}, {'key': 'TX', 'doc_count': 27}, {'key': 'AL', 'doc_count': 25}, {'key': 'MD', 'doc_count': 25}, {'key': 'TN', 'doc_count': 23}, {'key': 'MA', 'doc_count': 21}, {'key': 'NC', 'doc_count': 21}, {'key': 'ND', 'doc_count': 21}, {'key': 'ME', 'doc_count': 20}, {'key': 'MO', 'doc_count': 20}]


## Another Aggregation

Python version of the following example that can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/_executing_aggregations.html) in the Elasticsearch getting started tutorial.  This example calculates the average account balance by state.

The example below corresponds to the following DSL syntax.

`curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'`


```python
q = Bool( \
         must = [Q('match_all')],
)
s.query = q
a1 = A('terms',field='state.keyword')
a2 = A('avg',field='balance')
s.aggs.bucket('group_by_state',a1).metric('average_balance',a2)
resp = s.execute(ignore_cache=True)
print( resp.aggregations['group_by_state'].buckets )
```

    [{'key': 'ID', 'doc_count': 27, 'average_balance': {'value': 24368.777777777777}}, {'key': 'TX', 'doc_count': 27, 'average_balance': {'value': 27462.925925925927}}, {'key': 'AL', 'doc_count': 25, 'average_balance': {'value': 25739.56}}, {'key': 'MD', 'doc_count': 25, 'average_balance': {'value': 24963.52}}, {'key': 'TN', 'doc_count': 23, 'average_balance': {'value': 29796.782608695652}}, {'key': 'MA', 'doc_count': 21, 'average_balance': {'value': 29726.47619047619}}, {'key': 'NC', 'doc_count': 21, 'average_balance': {'value': 26785.428571428572}}, {'key': 'ND', 'doc_count': 21, 'average_balance': {'value': 26303.333333333332}}, {'key': 'ME', 'doc_count': 20, 'average_balance': {'value': 19575.05}}, {'key': 'MO', 'doc_count': 20, 'average_balance': {'value': 24151.8}}]


## Odds and Ends

Index size, pagination, sorting, limiting the fields returned by Elasticsearch.


```python
# A query
q = Bool( \
         must = [Q('match_all')],
)
s.query = q
resp = s.execute(ignore_cache=True)

# Size of the result
print( s.count() )

# Pagination
s1 = s[0:10]
resp = s1.execute()
print( 'First 10 Elements')
for hit in resp :
    print(hit)
s1 = s[10:20]
resp = s1.execute()
print( 'Next 10 Elements')
for hit in resp :
    print(hit)

# Sorting
s = s.sort('account_number')
s1 = s[0:10]
resp = s1.execute()
print( 'After Sorting - First 10 Elements')
for hit in resp :
    print(hit)
    
# Limit the fields returned by elasticsearch
s=s.source(['account_number','balance'])
resp = s.execute()
print( 'After Limiting Returned Fields')
for hit in resp :
    print(hit)
```

    1000
    First 10 Elements
    <Hit(bank/account/25): {'account_number': 25, 'balance': 40540, 'firstname': 'Virgi...}>
    <Hit(bank/account/44): {'account_number': 44, 'balance': 34487, 'firstname': 'Aurel...}>
    <Hit(bank/account/99): {'account_number': 99, 'balance': 47159, 'firstname': 'Ratli...}>
    <Hit(bank/account/119): {'account_number': 119, 'balance': 49222, 'firstname': 'Lave...}>
    <Hit(bank/account/126): {'account_number': 126, 'balance': 3607, 'firstname': 'Effie...}>
    <Hit(bank/account/145): {'account_number': 145, 'balance': 47406, 'firstname': 'Rowe...}>
    <Hit(bank/account/183): {'account_number': 183, 'balance': 14223, 'firstname': 'Huds...}>
    <Hit(bank/account/190): {'account_number': 190, 'balance': 3150, 'firstname': 'Blake...}>
    <Hit(bank/account/208): {'account_number': 208, 'balance': 40760, 'firstname': 'Garc...}>
    <Hit(bank/account/222): {'account_number': 222, 'balance': 14764, 'firstname': 'Rach...}>
    Next 10 Elements
    <Hit(bank/account/227): {'account_number': 227, 'balance': 19780, 'firstname': 'Cole...}>
    <Hit(bank/account/253): {'account_number': 253, 'balance': 20240, 'firstname': 'Meli...}>
    <Hit(bank/account/260): {'account_number': 260, 'balance': 2726, 'firstname': 'Kari'...}>
    <Hit(bank/account/265): {'account_number': 265, 'balance': 46910, 'firstname': 'Mari...}>
    <Hit(bank/account/335): {'account_number': 335, 'balance': 35433, 'firstname': 'Vera...}>
    <Hit(bank/account/366): {'account_number': 366, 'balance': 42368, 'firstname': 'Lydi...}>
    <Hit(bank/account/385): {'account_number': 385, 'balance': 11022, 'firstname': 'Rosa...}>
    <Hit(bank/account/397): {'account_number': 397, 'balance': 37418, 'firstname': 'Leon...}>
    <Hit(bank/account/400): {'account_number': 400, 'balance': 20685, 'firstname': 'Kane...}>
    <Hit(bank/account/450): {'account_number': 450, 'balance': 2643, 'firstname': 'Bradf...}>
    After Sorting - First 10 Elements
    <Hit(bank/account/0): {'account_number': 0, 'balance': 16623, 'firstname': 'Bradsh...}>
    <Hit(bank/account/1): {'account_number': 1, 'balance': 39225, 'firstname': 'Amber'...}>
    <Hit(bank/account/2): {'account_number': 2, 'balance': 28838, 'firstname': 'Robert...}>
    <Hit(bank/account/3): {'account_number': 3, 'balance': 44947, 'firstname': 'Levine...}>
    <Hit(bank/account/4): {'account_number': 4, 'balance': 27658, 'firstname': 'Rodriq...}>
    <Hit(bank/account/5): {'account_number': 5, 'balance': 29342, 'firstname': 'Leola'...}>
    <Hit(bank/account/6): {'account_number': 6, 'balance': 5686, 'firstname': 'Hattie'...}>
    <Hit(bank/account/7): {'account_number': 7, 'balance': 39121, 'firstname': 'Levy',...}>
    <Hit(bank/account/8): {'account_number': 8, 'balance': 48868, 'firstname': 'Jan', ...}>
    <Hit(bank/account/9): {'account_number': 9, 'balance': 24776, 'firstname': 'Opal',...}>
    After Limiting Returned Fields
    <Hit(bank/account/0): {'account_number': 0, 'balance': 16623}>
    <Hit(bank/account/1): {'account_number': 1, 'balance': 39225}>
    <Hit(bank/account/2): {'account_number': 2, 'balance': 28838}>
    <Hit(bank/account/3): {'account_number': 3, 'balance': 44947}>
    <Hit(bank/account/4): {'account_number': 4, 'balance': 27658}>
    <Hit(bank/account/5): {'account_number': 5, 'balance': 29342}>
    <Hit(bank/account/6): {'account_number': 6, 'balance': 5686}>
    <Hit(bank/account/7): {'account_number': 7, 'balance': 39121}>
    <Hit(bank/account/8): {'account_number': 8, 'balance': 48868}>
    <Hit(bank/account/9): {'account_number': 9, 'balance': 24776}>
