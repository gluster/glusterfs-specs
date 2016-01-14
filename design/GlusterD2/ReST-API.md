## Overview
The main communication interface with GlusterD-2.0 (GD2) will be a HTTP ReST interface. This interface will be made use of by the GD2 CLI, and will also be available for external consumers to use.

GD2 will provide ReST interfaces for the management of peers, management of volumes, local-GlusterD management, interfaces for monitoring (events) and long-running asynchronous operations.

## Authentication
The API will use a stateless authentication model using JSON web tokens. This will be based on the [Heketi authentication model][1].
> This is still tentative

## API
This is the first version of the GD2 ReST API, APIVERSION is **1**. All the endpoints defined below will be prefixed with `/v<API_VERSION>` unless otherwise mentioned.

GD2 uses JSON as its data serialization format. XML support is not planned initially.

Most APIs use the following methods on the URIs:
- URIs in the form of `/<version>/<ReST endpoint>/{id}`
- Requests and responses in JSON format
- `POST`: Send data to GD2 where the body has data described in JSON format.
- `GET`: Retrieve data from GD2 where the body has data described in JSON format.
- `DELETE`: Deletes the specified object from GD2.
- The HTTP content-type for all requests and responses will be `application/json`.

### Meta
The APIs are used to get some meta information about GD2 and the cluster itself.

#### Get version
Returns the GlusterD and API versions.
- **Method** : `GET`
- **Endpoint** : `/version` _will not be prefixed_
- **Request** : _Empty_
- **Response** :
	- **Status code** : `200 OK`
	- **Body** :
		- *glusterd-version* : A string. The GlusterD-2.0 version
		- *api-version* : A string. The ReST API version
		- *Example* :
```json
{
    "glusterd-version": "dev",
    "api-version": "1"
}
```

#### Get info
Returns some information about the cluster
- **Method** : `GET`
- **Endpoint** : `/info` _will not be prefixed_
- **Request** : _Empty_
- **Response** :
  - **Status code** : `200 OK`
  - **Body** : ***TBD***

> NOTE: The  APIs described only define the responses of successful request. Failed requests will follow a common response format, and have been defined later.

### Peers
The _peers_ endpoint will be used to manage peers in the cluster. All _peers_ endpoints will have the prefix `/peers/`.

#### Attach peer
- **Method** : `POST`
- **Endpoint** : `/peers/`
- **Request** :
	- **Parameters** :  _None_
	- **Body**:
	    - *addresses* : An array of strings. Gives a list of addresses by which the new host can be contacted. The addresses can be FQDNs, short-names or IP addresses. At least 1 address is required
		- *name* : A string, optional. The name to be used for the peer. This name can be used to refer to the peer in other commands. If not given, the first address in the addresses array will be used as the name.
		- Example :
```json
{
    "addresses": [
        "host1name1",
        "host1name2"
    ],
    "name": "host1"
}
```
- **Response** :
	- **Status code**: `201 Created`
	-  **Body**:
		- *id* : A string. The UUID of the newly added peer
		- Example :
```json
	{ "id" : "de305d54-75b4-431b-adb2-eb6b9e546014"}
```

#### Get peer
- **Method** : `GET`
- **Endpoint** : `/peers/{id}`
- **Request**:
	- **Parameters** :
		- `id` : UUID of the peer or the name of the peer to get.
	- **Body** : *Empty*
- **Response**:
	- **Status code**: `200 OK`
	- **Body** :
		- *id* : A string. The UUID of the peer
		- *name* : A string. The name of the peer.
		- *addresses* : An array of strings. Each entry in the list is an address by which the peer can be connected to.
		- *online* : A boolean. Gives online status of the peer
		- Example :
```json
{
    "id": "c1cf34e2-5bb7-4885-b8c2-0d92199993f9",
    "name": "peer2",
    "addresses": [
      "p2a1",
      "p2a2",
      "p2a3"
    ],
    "online": true
}
```

#### List peers
- **Method** : `GET`
- **Endpoint** : `/peers/`
- **Request**:
	- **Parameters** : *None*
	- **Body** : *Empty*
- **Response**:
	- **Status code** : `200 OK`
	- **Body** : An array of peer information JSON objects. For details of the peer information object refer *Get peer*.

#### Detach peer
- **Method** : `DELETE`
- **Endpoint**: `/peers/{id}`
- **Request**:
	- **Parameters** :
		- `id` : Name or ID of peer to be detached from the cluster
	- **Body** : *Empty*
- **Response**:
	- **Status code** : `204 No Content`
	- **Body** : *Empty*

### Volumes
The _volumes_ endpoints will be used to manage volumes in the cluster. All _volumes_ endpoints have the prefix `/volumes/`

#### Create volume
- **Method** : `POST`
- **Endpoint**: `/volumes/`
- **Request**:
	- **Parameters** : *None*
	- **Body** :
	    - *name* : A string. Name of the volume
	    - *stripe* :  An integer, optional. Represents the stripe count for a stripe volume
	    - *replica* : An integer, optional. Represents the replica count for a replicate volume. Default to be 1
	    - *arbiter* : An integer, optional. Represents the arbiter count for an arbiter volume
	    - *disperse-data* : An integer, optional. Represents the disperse-data count for a dispersed volume
	    - *disperse-redundancy* : An integer, optional. Represents the redundancy count for a dispersed volume
	    - *transport* : A string, optional. Represents the transport type (TCP/RDMA/Both). Default to be TCP
	    - *bricks* : An array of strings. Holds list of bricks to be configured for the volume. The semantics of a brick is maintained in the form of IP:brickpath
	    - *flags* : A JSON object with string-boolean key-value pairs, optional. The keys are flags and the values are their state. The default value is false.
        - Available flags
          - "reuse_bricks"
          - "allow_root_dir"
          - <need to add others we introduce>
    - Example :
```json
{
    "name": "test-vol",
    "replica" : 2,
    "bricks": [
        "x.x.x.x:/home/bricks/b1",
        "y.y.y.y:/home/bricks/b2"
    ],
}
```
- **Response**:
	- **Status code** : `201`
	- **Body** : A JSON object of a volume. For details please refer *Get volume*

#### Get volume
- **Method** : `GET`
- **Endpoint**: `/volumes/{id}`
- **Request**:
	- **Parameters** :
		- `id` : ID of the volume. The name of the volume is also accepted.
	- **Body** : *Empty*
- **Response**:
	- **Status code**: `200 OK`
	- **Body** :
	    - *id* : A string. UUID of the volume
	    - *name* :  A string. Name of the volume
	    - *type* : A string. Type of the volume which is any of the following
	        - Distribute
	        - Stripe
	        - Striped-Replicate
	        - Disperse
	        - Tier
	        - Distributed-Stripe
	        - Distributed-Replicate
	        - Distributed-Striped-Replicate
	        - Distributed-Disperse
	    - *stripe* :  An integer. Represents the stripe count for a stripe volume
	    - *replica* : An integer. Represents the replica count for a replicate volume
	    - *arbiter* : An integer. Represents the arbiter count for an arbiter volume
	    - *disperse-data* : An integer. Represents the disperse-data count for a dispersed volume
	    - *disperse-redundancy* : An integer. Represents the redundancy count for a dispersed volume
	    - *transport* : A string, transport type of the volume
	    - *options* : A map of key value strings. Represents volume tunables
	    - *status* : A string. Status of the volume either "Created" or "Started" or "Stopped"
	    - *version* : An integer. Version of the volume, internal to GlusterD
	    - *checksum* : An integer. md5 checksum value of the volume configuration, internal to GlusterD
	    - *bricks* : An array of strings. Represents each brick with following details:
	        - *id* : A string. UUID of the peer to which this brick belongs
	        - *hostname* : A string. Host/IP of the brick
	        - *path* : A string. Path of the brick
	    - Example :
```json
{
    "id": "0bef87b3-82ba-11e5-9d59-3c970e9eb10d"
    "name": "test-vol",
    "type": "Distribute",
    "stripe": 0,
    "replica": 0,
    "arbiter": 0,
    "disperse-data": 0,
    "disperse-redundancy": 0,
    "transport": "TCP"
    "options": {},
    "status": "Created",
    "checksum": 12345,
    "version": 1,
    "bricks": [
        {
            "id": "e5c26603-82bf-11e5-9d59-3c970e9eb10d",
            "hostname": "x.x.x.x"
            "path": "/gluster/brick1",
        },
        {
            "id": "06c47cc3-82c0-11e5-9d59-3c970e9eb10d",
            "hostname": "y.y.y.y"
            "path": "/gluster/brick2",
        }
    ]
}
```
#### Get volumes
- **Method** : `GET`
- **Endpoint**: `/volumes/`
- **Request** :
	- **Parameters** : *None*
	- **Body** : *Empty*
- **Response** :
	- **Status code** : `200 OK`
	- **Body** : A string-string JSON map, with volume-id as key and volume name as value.
    - Example :
```json
{
  "0bef87b3-82ba-11e5-9d59-3c970e9eb10d": "test-vol",
  "eb6aaea1-1689-4a45-8dbf-d4c98f76d9c5": "vol1"
}
```


#### Start volume
- **Method** : `POST`
- **Endpoint**: `/volumes/{name}/start`
- **Request** :
	- **Parameters** :
		- `name`: Name of the volume
	- **Request body** : _Empty_
- **Response** :
	- **Status code** : `200 OK`
	- **Response body** : _Empty_

#### Stop volume
- **Method** : `POST`
- **Endpoint**: `/volumes/{name}/stop`
- **Request** :
	- **Parameters** :
		- `name` : Name or ID of volume
	- **Body** : _Empty_
- **Response** :
	- **Status code** : `200 OK`
	- **Body** : _Empty_

#### Delete volume
- **Method** : `DELETE`
- **Endpoint**: `/volumes/{name}`
- **Request** :
	- **Parameters** :
		- `name` : Name of the volume
	- **Body** : _Empty_
- **Response** :
	- **Status code** : `204 No Content`
	- **Body** : _Empty_

[1]: https://github.com/heketi/heketi/wiki/API#authentication

