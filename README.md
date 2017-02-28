# HTTP DataService Protocol
The HTTP DataService protocol defines a mapping schema between the HTTP protocol and operations on data records.

## HTTP Resources and Data Records
The target of HTTP methods are HTTP resources and they are uniquely identified by their URL. In terms of data we distinguish data records that must be uniquely identifiable by some of their properties (fields) and logically grouped collections of these. HTTP resources do not imply singularity or multiplicity per se so first of all we introduce these data records concept to the HTTP world.

### Entity Set (Collection)
Entity Set is the HTTP Data Service concept for a collection of data records that is uniquely identifiable by a URL. In terms of relational database that would be Table or View. In terms of HTTP that is yet another resource. Each Entity Set is serviced by an HTTP Data Service. Requests to the URL of such HTTP Data Service (without further path segments) are requests for the data records related to this Entity Set.

The URL addressing pattern for Entity Sets is `{path-to-service}/(entity-set-data-service}`

Example:  `GET /services/js/forum/topics.js` returns the records for the `forums` Entity Set.

### Entity
Entity is the HTTP Data Service concept for a single data record that is uniquely identifiable by a URL. As data records are always part of a data records collection represented by an Entity Set, request paths always start with the URL to the corresponding Entity Set, followed by a segment identifying the entity. 

The URL addressing pattern for Entity Sets is `{path-to-service}/(entity-set-data-service}/{id}`

Example:  `GET /services/js/forum/topics.js/1` returns a data record entity from the `forums` Entity Set with identity value `1`.

### Association
Normally, entities are related to each other by Associations. In the relational models these are relations. In graph models this is arcs. Conceptually, these are all Associations. An Association is characterized by multiplicity, type, and direction. An Association always has source Entity that defines the context for it. Starting from that Entity one can navigate to the Association target or targets depending on the multiplicity. If the target multiplicity is _one_, the expected result is one Entity representing a data record in relation to the context Entity (by means of a foreign key equal to its data record id). If the target is _many_, the expected result is an Entity Set representing the data records with relation to the context entity data record. In relational terms this is an Entity Set of records that have foreign key equal to the id field of the context Entity data record. Associations of type many-to-many require an intermediate Entity Set to join the two related Entity Sets. 
If not specified otherwise, dependencies of type one-to-many are of type composition, i.e. cascading implicitly. That is, if you delete the "one" Entity, the associated "many" Entity Set Entities are deleted too.
The URL address of an association follows the identity path segment of a context Entity with a path segment for the name of the Association.

The URL addressing pattern for Entity Association is `{path-to-service}/(entity-set-data-service}/{id}/{association-name}`

Example:  `GET /services/js/forum/topics.js/1/comments` returns an array of data records from the `comments.js` Entity Set that are associated by a join key to the topic Entity with id `1` .

## Operations on Entity Sets
### List Entities
Sends a GET request to an Entity Set resource to fetch data records for this Entity Set and return an array of JSON formatted Entities for the data records found and HTTP code 200. In SQL that would be `SELECT * FROM EntitySet`
<pre>
Request
  GET {path-to-service}/(entity-set-data-service}
Response: 
  Code: 200 OK
  'X-dservice-list-count' Header: {Count number of the Entity objects in this Entity Set}
  Body: [array of JSON formatted entities]
</pre>
_Example_: 
<pre>
GET http://host/services/js/forum.js

200 OK
[]

With an SQL backend that translates to: SELECT * FROM FORUM
</pre>

#### Query operators
Normally, queries to Entity Sets are far more tailored for example to support paging and filter results by different criteria. HTTP Data Service defines standard Query operators and rules to tailor the data record queries. They are provided as URL query parameters.

The standard query operators are:
- **$limit**: adds a constraint to the query to limit the returned records to the provided number 

- **$offset**: adds a constraint to the query to return the records form a list with offset matching the provided number

- **$sort**: sorts the returned Entity Set by the property name matching this parameter value. The default sort order (if $order is not provided) is ascending. The value must be a valid Entity property name. Only one is supported.

- **$order**: defines the sort order. This is a valid parameter only if $sort parameter is provided. valid values are 'asc' or'desc'

- **$select**: instructs the query to include only the listed entity properties in the response. Valid values are none (interpreted as select $all),$all, one or more valid property names as defined by this service ORM backend /todo: make this service metadata instead to decouple/. Example: `myService.js?$select=name`, which will result in sql such as: `SELECT NAME FROM`(provided the name property maps to NAME in sql), and the returned entity will contain only the specified field' `{ name: ..}`. Note that if used together with $expand the returned entity will also include the ids used to join the expanded entities. 

- **$filter** (_beta_): instructs the query to use LIKE operator for the provided fields instead of = . Valid value is a list of valid entity persistent field names with type String. Always goes in pair with a key-value query string entries for the fields included in $filter. The argument for the LIKE operation is the value of this pair with % wildcard appended at the end. Useful for clients using suggestions such as type-ahead UI controls. Example:
`myservice.js?$filter=name&name=Shtu`, which will result in sql such as `NAME LIKE 'Shtu%'` (provided the name property maps to NAME in sql)

- **$expand**: instructs the query to expand in the response each entity's associated entity sets if any. Valid values are none (interpreted as expand $all),$all one or more valid association names as defined by this service ORM backend /todo: make this service metadata instead, to decouple/. Example: `myService.js?$expand=contacts`, which will expand inline in the returned entity the associated entities for contacts.

#### Property filters
Similar to the standard query operators, the query can be further tailored by providing filters on the selected Entities property names. They are provided as query parameters too. A query parameter with name equal to the name of a valid Entity property name and value translates to query filter for the data records. Multiple property filters can be added, each as a query string parameter, and they are translated to a boolean expression using boolean AND to concatenate filters. In terms of SQL that would be a filter in the `WHERE` clause: `SELECT * FROM EntitySet WHERE propertyName1=value1 AND propertyName2=value2`.

Example:
<pre>
GET /services/js/forum/topics.js?name=Test&status=3

With an SQL backend that translates to: SELECT * FROM FORUM WHERE NAME='Test' AND STATUS=3
</pre>

### Get Entity
Sends a GET request to an Entity Set resource to fetch data records for this Entity Set and return an array of JSON formatted Entities for the data records found and HTTP code 200. In SQL that would be `SELECT * FROM EntitySet`. Some of the standard query operators and property filters can be used to further constrain the query. The supported standard query operators are: $expand, $select and $filter.
<pre>
Request
GET {path-to-service}/(entity-set-data-service}/{id}
Response:
- Entity Found 
  Code: 200 OK
  Body: JSON-formatted object representing an Entity for the identified data record
- Entity Not Found
  Code: 404 Not Found
  Body: none
</pre>
_Example_: 
<pre>
GET http://host/services/js/forum.js/1

200 OK
{
 "id": 1,
 "name": "my topic",
 "status": 3
}

With an SQL backend that translates to: SELECT * FROM FORUM WHERE ID=1
</pre>

### Create Entity | Entities
Creates a new Entity or a set of Entities in a corresponding Entity Set. Valid inline associations provided with each Entity are inserted after its insert operation completes successfully and with references set to the id of the inserted context Entity. An attempt to delete created objects and rollback changes is performed on failure to create any of the inline associated entities or any of the entities in case of array argument.

* Response for single entity argument:

    No response body payload. The HTTP response code is `204 NO CONTENT`. A response header `Location` with value the url of the created entity.
* Response for multiple entities argument:

    The response body payload is a JSON formatted array of ids of the created entities. The HTTP response code is `200 OK`. 
 
<pre>
Request
  POST {path-to-service}/(entity-set-data-service}
  Body: A JSON formatted single entity or an array of entities to insert into this Entity Set. The Entity can contain inline dependent Entities for cascade insert.
Response:
  Code: 204 NO_CONTENT (for single entity input) | 200 OK (for multiple entities input)
  Location Header: {path-to-service}/(entity-set-data-service}/{id} (valid only for single entity input)
</pre>

### Count Entities
Returns the count of Entity objects in an Entity Set.

<pre>
Request
  GET {path-to-service}/(entity-set-data-service}/count
Response:
  Body: {
          count: {number of Entity object in Entity Set}
        }
Code: 200 OK
</pre>

## Operations on Entity
### Get Entity association
Returns the Entity or Entity Sets (depending on multiplicity) associated with this Entity.
<pre>
Request
  GET {path-to-service}/(entity-set-data-service}/{id}/{association-name}
Response:
  Code: 200 OK
  Body: a JSON object/array (depending on the multiplicity defined for this association) representing the requested entity's association set/entity
</pre>

Example: `GET http://host/service/Customers/190/orders`

### Remove
Deletes the requested Entity and all dependent Entities.
<pre>
Request
  DELETE {path-to-service}/(entity-set-data-service}/{id}
Response:
  - Existing Entity and successful delete operation
    Code: 204 OK
  - Non-existing Entity
    Code: 404 NOT FOUND
</pre>

Example: `DELETE http://host/service/Customers/190`

### Update
Deletes the requested Entity and all dependent Entities.
<pre>
Request
  PUT {path-to-service}/(entity-set-data-service}/{id}
  Body: JSON representing the entity to update.
Response:
  - Existing Entity and successful update operation
    Code: 204 OK
  - Non-existing Entity
    Code: 404 NOT FOUND
</pre>

Example: `PUT http://host/service/Customers/190`
