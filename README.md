# protoc-gen-openapi

Contains a protoc plugin that generates openapi v3 documents

**Forked from [github.com/kollalabs/protoc-gen-openapi](https://github.com/kollalabs/protoc-gen-openapi)**

Installation:

    go install github.com/wolaiba/protoc-gen-openapi@latest

Usage:

    protoc sample.proto -I. --openapi_out=version=1.2.3:.

## Testing

To see output during tests use `log.Print*`

```
go clean -testcache && go test
```

## Added Features
We have added some features that the Gnostic team most likely doesn't want to add :-)
Some are fairly Kolla specific, sorry. We try to hide Kolla specific functionality
in a way that won't trip anyone up.

* [Better Enum Support](#better-enum-support)
* [Summary Field](#summary-field)
* [Validation (protoc-gen-validate)](#validation)
* [Google Field Behavior Annotations](#google-field-behavior-annotations)
* [OAS3 header support](#oas3-header-support)
* [OpenAPI v3 Property Annotations](#openapi-v3-property-annotations)

### Better Enum Support
Enums work better by using string values of proto enums instead of ints.

### Summary Field

Sometimes you want more control over certain properties in the OpenAPI manifest. In our
case we wanted to use the `summary` property on routes to look nice for generating
documentation from the OpenAPI manifest. Normally the summary comes simply from the
name of the route. We added a feature that parses the comment over the proto service
method and looks for a pipe character ("`|`") and if it sees it, it will take anything to
the left of it and put it in the `summary` field, and anything to the right of it will
be the `description`. If no pipe is found it puts the whole comment in the description
like normal. From `/examples/tests/summary/message.proto`:

```proto
service Messaging {
    // Update Message Summary | This function updates a message.
    rpc UpdateMessage(Message) returns(Message) {
        option(google.api.http) = {
            patch: "/v1/messages/{message_id}"
            body: "text"
        };
    }
}
```

It generates the following OpenAPI:

```yaml
#...
paths:
    /v1/messages/{message_id}:
        patch:
            tags:
                - Messaging
            summary: Update Message Summary # Look at this beautiful summary...
            description: This function updates a message.
#...
```

### Validation

We added partial support for `protoc-gen-validate` annotations

OpenAPI spec allows for a small handful of input validation configurations.
Proto has an awesome plugin called `protoc-gen-validate` for generating validation code in
Go, Java, C++, etc. We took those same annotations and added support in this project
for them.

Usage: add `validate=true` to protoc command.

`protoc sample.proto -I. --openapi_out=version=1.2.3,validate=true:.`

#### Example

```proto
message Message {
    string message_id = 1;
    string text = 2 [(validate.rules)= {
        string: {
            uri:true,
            max_len:45,
            min_len:1
        }
    }];
    int64 mynum = 3 [(validate.rules).int64 = {gte:1, lte:30}];
}

```

outputs:

```yaml
components:
    schemas:
        Message:
            properties:
                message_id:
                    type: string
                text:
                    maxLength: 45
                    minLength: 1
                    type: string
                    format: uri
                mynum:
                    maximum: !!float 30
                    minimum: !!float 1
                    type: integer
                    format: int64

```

#### Supported Validators

String
- uri
- uuid
- email
- ipv4
- ipv6
- max_len
- min_len

Int32
- gte
- lte

Int64
- gte
- lte

Adding more can easily be done in the function `addValidationRules` in `/generator/openapi-v3.yaml`

### Google Field Behavior Annotations

* `(google.api.field_behavior) = REQUIRED` will add the field to the required list in the openAPI schema
* `(google.api.field_behavior) = OUTPUT_ONLY` will add the `readOnly` property to the field
* `(google.api.field_behavior) = INPUT_ONLY` will add the `writeOnly` property to the field
* TODO: `(google.api.field_behavior) = IMMUTABLE` will add the `x-createOnly` property to the field (not supported by openapi yet)

### OAS3 header support

TODO: Document this feature.

### OpenAPI v3 Property Annotations

Support for `openapi.v3.property` annotations to add `title` and `description` to schema fields.

#### Example

```proto
import "openapi/v3/annotations.proto";

message CreateUserRequest {
    string name = 1 [
        (openapi.v3.property) = {
            title: "User Name",
            description: "The display name of the user"
        }
    ];
    string email = 2 [
        (openapi.v3.property) = {title: "Email Address"}
    ];
}
```

outputs:

```yaml
components:
    schemas:
        CreateUserRequest:
            properties:
                name:
                    title: User Name
                    description: The display name of the user
                    type: string
                email:
                    title: Email Address
                    type: string
```

This is useful for:
- Generating better API documentation with field-level descriptions
- Providing localized field names for frontend validation error messages
- Adding context to auto-generated client code

