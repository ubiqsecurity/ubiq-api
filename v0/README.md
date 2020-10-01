## Ubiq Platform REST API (Version 0)

Version 0 of the API is available at `/api/v0/...`. The documentation below
omits the `/api/v0/` part of the path as it is common to all requests.

## Signing Requests

Amazon specifies a mechanism for signing its messages which is similar to the
drafts linked below but takes advantage of headers using the `x-aws-` naming
scheme. However, the `x-` naming scheme was deprecated in 2012 by
[RFC 6648](https://tools.ietf.org/html/rfc6648). The draft
[here](https://tools.ietf.org/html/draft-ietf-httpbis-message-signatures-00)
specifies a mechanism for signing HTTP messages and builds upon the draft
[here](https://tools.ietf.org/html/draft-cavage-http-signatures-12). The Ubiq
platform uses the signing scheme described below, which draws heavily from the
linked drafts with the goal of maintaining compatibility with updated versions
of the drafts under the assumption that they will eventually become
standardized.

In order to sign a message, the HTTP message must include the `Signature`
header as defined by the draft. The following parameters are required:
  - `keyId` (the client's public API key)
  - `headers`
  - `signature`

The `headers` parameter must specify the following headers (at a minimum),
which implies that those headers must be present in the HTTP message:
  - `(request-target)`
  - `Content-Type` (if present)
  - `Date`
  - `Digest` (even if payload is empty)
  - `Host`

## Responses

Responses containing data specific to various requests are described alongside
the particular request. Responses to unsuccessful requests, in general, will
take the form:
```http
HTTP/1.1 NNN Error
Content-Type: application/json

{
    "message": "Human-readable error message"
}
```

## Requests

### Create Data Encryption Key

- **URL**: `encryption/key`
- **Method**: `POST`
- **Authentication**: The request must be signed as described above
- **Body**:
    ```json
    {
        "uses": N
    }
    ```
    **Members**:
    - `uses` (Integer): The number of separate encryptions that will take
                        place under this key
- **Responses:**
    - `201 Created`
        ```json
        {
            "encrypted_private_key": "...",
            "encryption_session" : "...",
            "key_fingerprint" : "...",
            "wrapped_data_key" : "...",
            "encrypted_data_key" : "...",
            "max_uses": N
            "security_model" :
               {
                 "algorithm" : "...",
                 "enable_data_fragmentation" : B
               }
        }
        ```
        **Members**:
        - `encrypted_private_key` (String):
                                  A PEM-encoded encryption of the client's
                                  private key. The PEM encoding will contain
                                  all the necessary information (other than
                                  the key) to decrypt the key.
        - `encryption_session` (String):
                                  A string that identifies this specifc
                                  encryption request.
        - `key_fingerprint` (String):
                                  A string that identifies the encrypted
                                  data key.  It does not provide any details
                                  regarding what is stored in the key, just an
                                  easy way to identify the key.
        - `wrapped_data_key` (String):
                                  A base64-encoded encryption of the data
                                  key encrypted using the public counterpart
                                  of the `encrypted_private_key` contained in 
                                  this message and the RSAES-OAEP padding scheme
                                  with SHA-1 as the underlying hash function.
        - `encrypted_data_key` (String):
                                  A base64 encoding of an opaque data structure.
                                  The data should be decoded and then carried
                                  with each ciphertext encrypted with the data
                                  key. It can later be used to retrieve the data
                                  key during decryption.
        - `max_uses` (Integer):
                                  The maximum number of times that this
                                  key may be used. This number is usually the
                                  same as the request but may be limited by the
                                  system or other security settings.
        - `security_model` (Object):
            - `algorithm` (String):
                                  An identifier describing the algorithm to
                                  be used with the encryption key
            - `enable_data_fragmentation` (Boolean):
                                  A boolean value indicating whether data
                                  fragmentation logic should be applied when
                                  encrypting data using this key.

### Update Data Encryption Key Usage

- **URL**: `encryption/key/:key_fingerprint/:encryption_session`
- **Method**: `PATCH`
- **Authentication**: The request must be signed as described above
- **Body**:
    ```json
    {
        "requested": N,
        "actual": M
    }
    ```
    **Members**:
    - `requested` (Integer): The number of uses of the key specified
                             during creation of the key
    - `actual` (Integer): The actual number of encryptions performed
                          using the key
- **Responses**:
    - `204 No Content`
- **Notes**:
    This call is not necessary if the number of encryptions requested and
    the number performed are equal.

### Decrypt Data Key

- **URL**: `decryption/key`
- **Method**: `POST`
- **Authentication**: The request must be signed as described above
- **Body**:
    The request body is a JSON object
    ```json
    {
        "encrypted_data_key": "..."
    }
    ```
    **Members**:
    - `encrypted_data_key` (String): 
                       The encrypted data key associated with the cipher text.
                       The content of this member is exactly equivalent to
                       that of the `encrypted_data_key` member returned from a
                       successful `POST` to `encryption/key`.
- **Responses**:
    - `200 OK`
        ```json
        {
            "encrypted_private_key": "...",
            "encryption_session" : "...",
            "key_fingerprint" : "...",
            "wrapped_data_key" : "...",
        }
        ```
        **Members**:
        - `encrypted_private_key` (String):
                                  A PEM-encoded encryption of the client's
                                  private key. The PEM encoding will contain
                                  all the necessary information (other than
                                  the key) to decrypt the key.
        - `encryption_session` (String):
                                  A string that identifies this specifc
                                  encryption request.
        - `key_fingerprint` (String):
                                  A string that identifies the encrypted
                                  data key.  It does not provide any details
                                  regarding what is stored in the key, just an
                                  easy way to identify the key.
        - `wrapped_data_key` (String):
                                  A base64-encoded encryption of the data
                                  key encrypted using the public counterpart
                                  of the `encrypted_private_key` contained in 
                                  this message and the RSAES-OAEP padding scheme
                                  with SHA-1 as the underlying hash function.

### Update Data Decryption Key Usage

- **URL**: `decryption/key/:key_fingerprint/:encryption_session`
- **Method**: `PATCH`
- **Authentication**: The request must be signed as described above
- **Body**:
    ```json
    {
        "uses": N
    }
    ```
    **Members**:
    - `uses` (Integer): The number of decryptions performed using the key
- **Responses**:
    - `204 No Content`
    
