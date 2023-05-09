# CUP2PY
#### Lucy Betts, Ilia Bolgov, Jodie Furnell, Annabel May
Proof-of-concept for decentralised protocol for peer-to-peer (P2P) communication.

## Theory

Theory behind the project and detailed description of the communication protocol is in this paper: **PAPER!!!**

## Functionality & Tutorial
This tutorial will show how to use CUP2PY to create and manage Address Books and User Records, how to create User Records from RSA public/private key pairs and generate the RSA key pairs. 
The library has the functionality to create and handle UPDATE and SEARCH requests, and convert them from string representations for communication to objects of dedicated classes and back. 
The library also has the functionality to encrypt and decrypt local files with RSA private keys using AES algorithm.

#### The library does not contain functions for the actual network communication: sending and receiving the request strings, as well as actual messaging functionality. A different library can be used for that, such that 'socket'.

### User Record

The class `UserRecord` has four attributes:
```
   publicKey, 
   ip, 
   updateDate, 
   digitalSignature
```

For the attribute `publicKey` class `rsa.PublicKey` is used. For its string representation 
function `publicKeyToString(publicKey)` returns `.PEM` version of the public key. To return it
back into `rsa.PublicKey` object function `stringToPublicKey(stringPublicKey)` is used.

Attribute `ip` is stored as a string. 

For the attribute `updateDate` class `datetime.datetime` is used. For its string representation 
format `%Y-%m-%d %H:%M:%S.%f` is used. To return it back into `datetime.datetime` object function
`datetime.datetime.strptime(stringUpdateDate, "%Y-%m-%d %H:%M:%S.%f")` is used.

Attribute `digitalSignature` is stored as bytes object, for its string representation - 
function `signatureToB64(signature)` returns Base64 encoded string. To return it back into
bytes object - function `b64ToSignature(b64Signature)` is used.

These attributes are stored as strings in Address Books (SQL tables).

### Creation of new session object

The library uses notion of sessions - this allows the user to choose a specific Address Book and User Record, and to create and handle requests using chosen Address Book and User Record without conflicts.

To create a new session, we need to use:
```python
import cup2py

s = cup2py.Session()
```

### Setting `DatabaseName` and `chosenAddressBook` of the session

Every session has these attributes:

```
        databaseName,
        chosenAddressBook,
        chosenUserRecord,
        chosenUserRecordPrivateKeyFile
```
All the attributes are set as `None` by default.

To set the values for `databaseName` and `chosenAddressBook` we will use dedicated function of the class Session.
The name of Address Book is the name of SQL table which will be used to represent it.

The `databaseName` can be a direct path where the database file should be created.

```python
s.setDatabaseName('<databaseName>.db')
s.setAddressBook('<addressBookName>')
```
To create an Address Book (SQL table) with chosen name:
```python
s.createAddressBookInDatabase()
```
Now we have created a database and an Address Book (SQL table) in it.

### Generation of User Records
The library allows to generate a new RSA keypair and create new object of a class `UserRecord` using this keypair
with function `generateUser(localName, ip, path=os.getcwd())`.

The keypair is going to be stored in current working directory by default, or in the chosen path, with names 
`<localName>_publicKey.pem` and `<localName>_privateKey.pem`.

```python
cup2py.generateUser('<localName>', '8.8.8.8')
```

We can also create a new User Record from already existing RSA keypair.
Here we are loading the keypair files `<localName>_publicKey.pem` and `<localName>_privateKey.pem` and creating new User Record from them with IP address `8.8.8.8`.
```python
pub, priv = cup2py.loadPublicKey('<localName>'), cup2py.loadPrivateKey('<localName>')
user = cup2py.userRecordFromKeyPair(priv, pub, '8.8.8.8')
```

We can also manually generate new RSA keypair:
```python
generatedPrivateKey, generatedPublicKey = cup2py.generateUserKeyPair()
```
### Set `chosenUserRecord` for the session
All requests contain a list of hashes of public keys of users,
which already received the request. This is done to minimise unnecessary
flood in the network.

To successfully add the public key hash of local User Record and
route UPDATE and SEARCH requests we set `chosenUserRecord` 
attribute for the session.

To set a User Record as default User Record for the session:
```python
s.setUser(someUserRecord)
```
### Get User Record from Address Book: by public key or by hash
To manually get a required User Record from Address Book by public key hash:
```python
user = s.getUserByHash(publicKeyHashValue)
```
or by public key object:
```python
user = s.getUser(publicKey)
```
### Updating Address Book
To update or add a new User Record in the Address Book of the session: 

```python
s.updateAddressBookRecord(someUserRecord)
```
This function will update the record, if there already is an older one with the same public key
in the Address Book, or add a new one otherwise.

This function verifies the digital signature of the record and returns the User Record
if it was updated, or `False` otherwise.

### New UPDATE Request
To create a new UPDATE request:
```python
someUpdateRequest = newUpdateRequest(someUserRecord, updateDepth=3)
```
This will generate new UPDATE request with given User Record and 'recursion depth' 3. 

### New SEARCH Request
To create a new SEARCH request:
```python
someSearchRequest = newSearchRequest(searchedPublicKeyHash, senderUserRecord, searchDepth=7)
```
This will generate new SEARCH request with given sender's User Record, searched public key hash and 'recursion depth' 7. 

### Prepare UPDATE Requests and SEARCH Requests for sending - string form and list of IP addresses
For transmission purposes, UPDATE and SEARCH requests can be represented in their string form.

**Standard form for UPDATE request:**
```
UPDATE;<SenderStringPublicKey>;<SenderIP>;<SenderUpdateDate>;<SenderStringDigitalSignature>;<RecursionDepth>;[<ListOfChechkedHashes>]
```
**Standard form for SEARCH request:**
```
SEARCH;<SearchedPublicKeyHash>;<SenderStringPublicKey>;<SenderIP>;<SenderUpdateDate>;<SenderStringDigitalSignature>;<RecursionDepth>;[<ListOfChechkedHashes>]
```

To get the string representation for UPDATE requests and list of IP addresses of users to send:
```python
someUpdateRequestToSend, someIpList = updateToSend(someUpdateRequest)
```

To get the string representation for SEARCH requests and list of IP addresses of users to send:
```python
someSearchRequestToSend, someIpList = searchToSend(someSearchRequest)
```

The functions return tuples in a form `(updateRequestToSend, ipList)` or `(searchRequestToSend, ipList)`
where `ipList` is a list containing all IP addresses stored in default Address Book, except for the once listed in request as "checked".

If there is a specific `ipList` user wants to use:
```python
anotherUpdateRequestToSend, anotherIpList = updateToSend(someUpdateRequest, ipList)
```
```python
anotherSearchRequestToSend, anotherIpList = searchToSend(someSearchRequest, ipList)
```
This way request can be generated even if the `chosenAddressBook` and `databaseName` hasn't been defined.

### Request Handling
To handle and process received UPDATE and SEARCH requests there is a function `requestHandler(message)`.
This function receives a request string, identifies the type of request (UPDATE or SEARCH) and processes it.

For UPDATE requests:
1) Converts given string into the object of class `UpdateRecord`
2) Checks digital signature of the User Record in the request; 
3) If digital signature is verified, updates User Record with given public key in local Address Book if the received User Record is newer than existing one
or adds a new one if there was no User Record with given public key;
4) If `updateDepth > 0`, generates updated UPDATE request with `updateDepth` decreased by 1 and hash of the `chosenUserRecord` of the current session added to `checkedPublicKeyHashes` 
   and returns a tuple with string representation of new request and list of IP addresses in a form `(updateRequestToSend, ipList)`;

Otherwise, returns `None`.

Example:
```python
handledUpdateRequest = s.requestHandler(f"UPDATE;"
                                 f"{stringPublicKey};"
                                 f"{stringIp};"
                                 f"{stringUpdateDate};"
                                 f"{stringDigitalSignature};"
                                 f"{stringUpdateDepth};"
                                 f"[{updatedHash_1}, ...]")
```

For SEARCH requests:
1) Converts given string into the object of class `SearchRecord`
2) Checks if `searchDepth > 0`
3) If it is, generates updated SEARCH request with `searchDepth` decreased by 1 and hash of `chosenUserRecord` of the current session added to `searchedPublicKeyHashes`
4) Checks if there is a User Record in `chosenAddressBook` with given `publicKeyHash`
5) If there is, returns a list in a form `[(updateRequestWithFoundUserRecord, [senderIp]), (searchRequestToSend, ipList)]`, where 
   first tuple contains UPDATE request string with requested User Record and list with IP of sender of SEARCH request and second tuple contains
   updated SEARCH request and list of all IP addresses it should be sent to (in case the record in the `chosenAddressBook` is not the newest available).
6) If requested User Record wasn't found, returns tuple in a form `(searchRequestToSend, ipList)`.

Otherwise, returns `None`.

Example:
```python
handledSearchRequest = s.requestHandler(f'SEARCH;'
                 f'{searchedHash};'
                 f'{stringSenferPublicKey};'
                 f'{stringSenderIp};'
                 f'{stringSenderUpdateDate};'
                 f'{stringSenderDigitalSignature};'
                 f'{stringSearchDepth};'
                 f'[{stringCheckedHash_1}, ...]')
```

###Encryption of local private key files

#### For further work with this library (sending and receiving the requests, chats between users) the use of network libraries (such as socket) is required.
