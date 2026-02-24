openapi: 3.1.0
info:
  title: ActiveMQ Artemis Management Plane API
  version: 1.1.0
  description: |
    CRUD and bulk operations for ActiveMQ Artemis (addresses, queues, diverts, security/access control, policies), with RACI enforced by Azure AD groups. Implementation mappings provided for Jolokia (HTTP-to-JMX).

x-raci:
  roles: [Admin, DataOwner, DataConsumer, Anyone]
  # Address namespace is the ownership boundary. The API determines a resource's namespace
  # via the `namespace` field (preferred) or by parsing a naming convention you define.
  ownership:
    scope: addressNamespace
    resourceField: namespace
    groupPatterns:
      Admin:        ["CN=MQ-Admins,OU=Groups,DC=example,DC=com"]
      DataOwner:    ["CN=MQ-DataOwners-{namespace},OU=Groups,DC=example,DC=com"]
      DataConsumer: ["CN=MQ-Consumers-{namespace},OU=Groups,DC=example,DC=com"]
      Anyone: []
  enforcement:
    groupsClaim: "groups"   # groups Object IDs/DNs from Azure AD ID token
    mode: "deny-by-default"
    precedence: ["Admin","DataOwner","DataConsumer","Anyone"]

x-impl:
  jolokia:
    # Adjust if you expose it at a different path (e.g., /api/jolokia)
    baseUrl: "https://{brokerHost}:8161/console/jolokia"
    # Artemis exposes management via JMX MBeans; Jolokia bridges HTTP<->JMX.
    # (Same capability surface used by the built-in Artemis web console.)
    notes: "All exec/read map to Artemis MBeans under org.apache.activemq.artemis.*" 

servers:
  - url: https://api.example.com/artemis-mgmt/v1

security:
  - oauth2_azure_ad: [broker.read, broker.write, broker.admin]

components:
  securitySchemes:
    oauth2_azure_ad:
      type: oauth2
      description: Azure AD client credentials. The access token must include a `groups` claim for RACI mapping.
      flows:
        clientCredentials:
          tokenUrl: https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token
          scopes:
            broker.read: Read-only operations
            broker.write: Create/Update/Delete operations
            broker.admin: Access-control and high-privilege operations
    idempotency_key:
      type: apiKey
      in: header
      name: Idempotency-Key

  parameters:
    PageParam:
      name: page
      in: query
      schema: { type: integer, minimum: 1, default: 1 }
    PageSizeParam:
      name: pageSize
      in: query
      schema: { type: integer, minimum: 1, maximum: 1000, default: 100 }
    SortParam:
      name: sort
      in: query
      schema: { type: string }
      description: Comma-separated sort (e.g., "name,-createdAt")
    FilterParam:
      name: filter
      in: query
      schema: { type: string }
      description: RSQL-like filter (e.g., "namespace==finance;durable==true")
    IfMatch:
      name: If-Match
      in: header
      schema: { type: string }
      description: ETag for conditional update/delete

  schemas:
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        detail: { type: string }
        instance: { type: string, format: uri }
        errors: { type: array, items: { type: string } }
        correlationId: { type: string }

    PageMeta:
      type: object
      properties:
        page: { type: integer }
        pageSize: { type: integer }
        totalItems: { type: integer }
        totalPages: { type: integer }

    BulkRequest:
      type: object
      required: [items]
      properties:
        items:
          type: array
          items: { type: object }

    BulkResultItem:
      type: object
      properties:
        index: { type: integer }
        status: { type: integer }
        id: { type: string }
        error: { $ref: "#/components/schemas/Problem" }

    BulkResult:
      type: object
      properties:
        results:
          type: array
          items: { $ref: "#/components/schemas/BulkResultItem" }

    Address:
      type: object
      required: [name]
      properties:
        id: { type: string }
        name: { type: string }
        namespace: { type: string, description: "Ownership namespace for RACI (e.g., 'finance')" }
        routingTypes:
          type: array
          items: { type: string, enum: [ANYCAST, MULTICAST] }
        autoCreateQueues: { type: boolean, default: false }
        autoDeleteQueues: { type: boolean, default: false }
        labels: { type: object, additionalProperties: { type: string } }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
        etag: { type: string }

    Queue:
      type: object
      required: [name, address]
      properties:
        id: { type: string }
        name: { type: string }
        namespace: { type: string }
        address: { type: string }
        fqqn: { type: string, description: "Fully-Qualified Queue Name: address::queue" }
        routingType: { type: string, enum: [ANYCAST, MULTICAST] }
        durable: { type: boolean, default: true }
        maxConsumers: { type: integer, default: -1 }
        exclusive: { type: boolean, default: false }
        purgeOnNoConsumers: { type: boolean, default: false }
        filter: { type: string, nullable: true }
        lastValue: { type: boolean, default: false }
        autoCreateAddress: { type: boolean, default: false }
        labels: { type: object, additionalProperties: { type: string } }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
        etag: { type: string }

    Divert:
      type: object
      required: [name, address, forwardingAddress]
      properties:
        id: { type: string }
        name: { type: string }
        namespace: { type: string }
        address: { type: string }
        forwardingAddress: { type: string }
        exclusive: { type: boolean, default: false }
        filter: { type: string, nullable: true }
        routingType: { type: string, enum: [ANYCAST, MULTICAST, PASS, STRIP], default: STRIP }
        transformerClass: { type: string, nullable: true }
        enabled: { type: boolean, default: true }
        labels: { type: object, additionalProperties: { type: string } }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
        etag: { type: string }

    SecurityPermissionSet:
      type: object
      description: "Map permissions to role/group names for an address match"
      properties:
        send: { type: array, items: { type: string } }
        consume: { type: array, items: { type: string } }
        browse: { type: array, items: { type: string } }
        manage: { type: array, items: { type: string } }
        createDurableQueue: { type: array, items: { type: string } }
        deleteDurableQueue: { type: array, items: { type: string } }
        createNonDurableQueue: { type: array, items: { type: string } }
        deleteNonDurableQueue: { type: array, items: { type: string } }

    SecuritySetting:
      type: object
      required: [match, permissions]
      properties:
        id: { type: string }
        match: { type: string, description: "Address match (supports # and * wildcards)" }
        permissions: { $ref: "#/components/schemas/SecurityPermissionSet" }
        labels: { type: object, additionalProperties: { type: string } }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
        etag: { type: string }

    AddressSettings:
      type: object
      required: [match]
      properties:
        id: { type: string }
        match: { type: string }
        deadLetterAddress: { type: string, nullable: true }
        expiryAddress: { type: string, nullable: true }
        maxDeliveryAttempts: { type: integer, default: 10 }
        maxSizeBytes: { type: integer, default: -1 }
        maxSizeMessages: { type: integer, default: -1 }
        addressFullMessagePolicy: { type: string, enum: [PAGE, DROP, BLOCK, FAIL], default: PAGE }
        pageSizeBytes: { type: integer, default: 10485760 }
        pageMaxCacheSize: { type: integer, default: 5 }
        redeliveryDelay: { type: integer, default: 0 }
        redeliveryMultiplier: { type: number, default: 1.0 }
        redistributionDelay: { type: integer, default: -1 }
        autoCreateQueues: { type: boolean, default: true }
        autoDeleteQueues: { type: boolean, default: true }
        autoCreateAddresses: { type: boolean, default: true }
        autoDeleteAddresses: { type: boolean, default: true }
        defaultLastValueQueue: { type: boolean, default: false }
        defaultExclusiveQueue: { type: boolean, default: false }
        slowConsumerThreshold: { type: integer, default: -1 }
        slowConsumerPolicy: { type: string, enum: [NOTIFY, KILL], default: NOTIFY }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
        etag: { type: string }

    Bridge:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        sourceQueue: { type: string }
        forwardingAddress: { type: string }
        routingType: { type: string, enum: [ANYCAST, MULTICAST], nullable: true }
        filter: { type: string, nullable: true }
        transformerClass: { type: string, nullable: true }
        enabled: { type: boolean, default: true }
        etag: { type: string }

    Connector:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        uri: { type: string }

    Acceptor:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        uri: { type: string }

    MessageMoveRequest:
      type: object
      required: [toAddress]
      properties:
        toAddress: { type: string }
        filter: { type: string, nullable: true }
        limit: { type: integer, default: -1 }

    PurgeRequest:
      type: object
      properties:
        filter: { type: string, nullable: true }
        limit: { type: integer, default: -1 }

    AuditRecord:
      type: object
      properties:
        id: { type: string }
        timestamp: { type: string, format: date-time }
        principal: { type: string }
        action: { type: string }
        resourceType: { type: string }
        resourceId: { type: string }
        before: { type: object }
        after: { type: object }

paths:
  # -------------------- Addresses --------------------
  /addresses:
    get:
      summary: List addresses
      operationId: listAddresses
      tags: [Addresses]
      parameters: [ { $ref: "#/components/parameters/PageParam" }, { $ref: "#/components/parameters/PageSizeParam" }, { $ref: "#/components/parameters/SortParam" }, { $ref: "#/components/parameters/FilterParam" } ]
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  items: { type: array, items: { $ref: "#/components/schemas/Address" } }
                  page: { $ref: "#/components/schemas/PageMeta" }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer, Anyone] }
      x-impl:
        jolokia:
          # Read via AddressControl MBeans or list from ActiveMQServerControl
          read:
            - mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
              attribute: 'AddressNames'   # or listAddresses()
  /addresses:
    post:
      summary: Create address
      operationId: createAddress
      tags: [Addresses]
      security: [ { oauth2_azure_ad: [broker.write] }, { idempotency_key: [] } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/Address" } } }
      responses:
        "201":
          description: Created
          headers: { ETag: { schema: { type: string } } }
          content: { application/json: { schema: { $ref: "#/components/schemas/Address" } } }
        "409":
          description: Conflict
          content: { application/problem+json: { schema: { $ref: "#/components/schemas/Problem" } } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'createAddress(java.lang.String,java.lang.String)'
            params:
              - '$.name'         # address name
              - 'routingTypes'   # e.g., "ANYCAST,MULTICAST"
          notes: "Choose the exact createAddress signature for your Artemis version; see ActiveMQServerControl javadoc."
  /addresses/bulk:
    post:
      summary: Bulk create addresses
      operationId: bulkCreateAddresses
      tags: [Addresses, Bulk]
      security: [ { oauth2_azure_ad: [broker.write] }, { idempotency_key: [] } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/BulkRequest"
                - type: object
                  properties:
                    items: { type: array, items: { $ref: "#/components/schemas/Address" } }
      responses:
        "207": { description: Multi-Status, content: { application/json: { schema: { $ref: "#/components/schemas/BulkResult" } } } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }

  /addresses/{id}:
    get:
      summary: Get address
      operationId: getAddress
      tags: [Addresses]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/Address" } } } }
        "404": { description: Not Found }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer, Anyone] }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Address,address={id},brokerName={brokerName}'
              attribute: '*'
    patch:
      summary: Update address (partial)
      operationId: updateAddress
      tags: [Addresses]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { $ref: "#/components/parameters/IfMatch" }, { name: id, in: path, required: true, schema: { type: string } } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/Address" } } }
      responses:
        "200": { description: OK, headers: { ETag: { schema: { type: string } } }, content: { application/json: { schema: { $ref: "#/components/schemas/Address" } } } }
        "412": { description: Precondition Failed }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }
    delete:
      summary: Delete address
      operationId: deleteAddress
      tags: [Addresses]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { $ref: "#/components/parameters/IfMatch" }, { name: id, in: path, required: true, schema: { type: string } } ]
      responses: { "204": { description: No Content } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'deleteAddress(java.lang.String)'
            params: ['{id}']

  # -------------------- Queues --------------------
  /queues:
    get:
      summary: List queues
      operationId: listQueues
      tags: [Queues]
      parameters: [ { $ref: "#/components/parameters/PageParam" }, { $ref: "#/components/parameters/PageSizeParam" }, { $ref: "#/components/parameters/SortParam" }, { $ref: "#/components/parameters/FilterParam" } ]
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  items: { type: array, items: { $ref: "#/components/schemas/Queue" } }
                  page: { $ref: "#/components/schemas/PageMeta" }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer, Anyone] }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
              attribute: 'QueueNames'
    post:
      summary: Create queue
      operationId: createQueue
      tags: [Queues]
      security: [ { oauth2_azure_ad: [broker.write] }, { idempotency_key: [] } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/Queue" } } }
      responses:
        "201": { description: Created, headers: { ETag: { schema: { type: string } } }, content: { application/json: { schema: { $ref: "#/components/schemas/Queue" } } } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            # Select the overload appropriate to your Artemis version. 
            operation: 'createQueue(java.lang.String,java.lang.String,java.lang.String,boolean)'
            params:
              - '$.address'        # address
              - '$.name'           # queue name
              - '$.routingType'    # ANYCAST|MULTICAST
              - '$.durable'        # true|false

  /queues/bulk:
    post:
      summary: Bulk create queues
      operationId: bulkCreateQueues
      tags: [Queues, Bulk]
      security: [ { oauth2_azure_ad: [broker.write] }, { idempotency_key: [] } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/BulkRequest"
                - type: object
                  properties:
                    items: { type: array, items: { $ref: "#/components/schemas/Queue" } }
      responses: { "207": { description: Multi-Status, content: { application/json: { schema: { $ref: "#/components/schemas/BulkResult" } } } } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }

  /queues/{id}:
    get:
      summary: Get queue
      operationId: getQueue
      tags: [Queues]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/Queue" } } } }
        "404": { description: Not Found }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer, Anyone] }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Queue,queue={id},brokerName={brokerName},address=*'
              attribute: '*'
    patch:
      summary: Update queue
      operationId: updateQueue
      tags: [Queues]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/IfMatch" } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/Queue" } } }
      responses:
        "200": { description: OK, headers: { ETag: { schema: { type: string } } }, content: { application/json: { schema: { $ref: "#/components/schemas/Queue" } } } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }
    delete:
      summary: Delete queue
      operationId: deleteQueue
      tags: [Queues]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/IfMatch" } ]
      responses: { "204": { description: No Content } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'destroyQueue(java.lang.String)'
            params: ['{id}']

  # Queue message ops (move/purge/count), implemented via QueueControl/ServerControl
  /queues/{id}/messages:count:
    get:
      summary: Count messages on queue
      operationId: countQueueMessages
      tags: [Queues, Messages]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/FilterParam" } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { type: object, properties: { count: { type: integer } } } } } }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer], namespaceScoped: true }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Queue,queue={id},brokerName={brokerName},address=*'
              attribute: 'MessageCount'

  /queues/{id}/messages:purge:
    post:
      summary: Purge messages on queue
      operationId: purgeQueueMessages
      tags: [Queues, Messages]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      requestBody:
        content: { application/json: { schema: { $ref: "#/components/schemas/PurgeRequest" } } }
      responses: { "202": { description: Accepted } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Queue,queue={id},brokerName={brokerName},address=*'
            operation: 'removeMessages(java.lang.String)'
            params: ['${body.filter:-}'] # empty string removes all

  /queues/{id}/messages:move:
    post:
      summary: Move messages from queue to address
      operationId: moveQueueMessages
      tags: [Queues, Messages]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/MessageMoveRequest" } } }
      responses: { "202": { description: Accepted } }
      x-raci: { allowed: [Admin, DataOwner], namespaceScoped: true }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Queue,queue={id},brokerName={brokerName},address=*'
            operation: 'moveMessages(java.lang.String,java.lang.String)'
            params: ['${body.filter:-}', '${body.toAddress}']

  # -------------------- Diverts --------------------
  /diverts:
    get:
      summary: List diverts
      operationId: listDiverts
      tags: [Diverts]
      parameters: [ { $ref: "#/components/parameters/PageParam" }, { $ref: "#/components/parameters/PageSizeParam" } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { type: object, properties: { items: { type: array, items: { $ref: "#/components/schemas/Divert" } }, page: { $ref: "#/components/schemas/PageMeta" } } } } } }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer] }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
              attribute: 'DivertNames'
    post:
      summary: Create divert
      operationId: createDivert
      tags: [Diverts]
      security: [ { oauth2_azure_ad: [broker.write] }, { idempotency_key: [] } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/Divert" } } }
      responses: { "201": { description: Created, content: { application/json: { schema: { $ref: "#/components/schemas/Divert" } } } } }
      x-raci: { allowed: [Admin], namespaceScoped: true }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            # Some versions accept a JSON divert configuration string
            operation: 'createDivert(java.lang.String)'
            params:
              - '${jsonSerialize(body)}'
          notes: "Diverts support exclusive, filter, and routingType STRIP/PASS/ANYCAST/MULTICAST; see Artemis docs."

  /diverts/bulk:
    post:
      summary: Bulk create/update diverts
      operationId: bulkUpsertDiverts
      tags: [Diverts, Bulk]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/BulkRequest"
                - type: object
                  properties:
                    items: { type: array, items: { $ref: "#/components/schemas/Divert" } }
      responses: { "207": { description: Multi-Status, content: { application/json: { schema: { $ref: "#/components/schemas/BulkResult" } } } } }
      x-raci: { allowed: [Admin], namespaceScoped: true }

  /diverts/{id}:
    get:
      summary: Get divert
      operationId: getDivert
      tags: [Diverts]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/Divert" } } } }
        "404": { description: Not Found }
      x-raci: { allowed: [Admin, DataOwner, DataConsumer], namespaceScoped: true }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Divert,divert={id},brokerName={brokerName}'
              attribute: '*'
    patch:
      summary: Update divert
      operationId: updateDivert
      tags: [Diverts]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/IfMatch" } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/Divert" } } }
      responses:
        "200": { description: OK, headers: { ETag: { schema: { type: string } } }, content: { application/json: { schema: { $ref: "#/components/schemas/Divert" } } } }
      x-raci: { allowed: [Admin], namespaceScoped: true }
    delete:
      summary: Delete divert
      operationId: deleteDivert
      tags: [Diverts]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/IfMatch" } ]
      responses: { "204": { description: No Content } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'destroyDivert(java.lang.String)'
            params: ['{id}']

  # -------------------- Access Controls (Security Settings) --------------------
  /security-settings:
    get:
      summary: List security settings
      operationId: listSecuritySettings
      tags: [AccessControl]
      parameters: [ { $ref: "#/components/parameters/PageParam" }, { $ref: "#/components/parameters/PageSizeParam" } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { type: object, properties: { items: { type: array, items: { $ref: "#/components/schemas/SecuritySetting" } }, page: { $ref: "#/components/schemas/PageMeta" } } } } } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
              attribute: 'Roles'  # or getRolesAsJSON(match) per ActiveMQServerControl
    post:
      summary: Create security setting
      operationId: createSecuritySetting
      tags: [AccessControl]
      security: [ { oauth2_azure_ad: [broker.admin] } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/SecuritySetting" } } }
      responses:
        "201": { description: Created, content: { application/json: { schema: { $ref: "#/components/schemas/SecuritySetting" } } } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'addSecuritySettings(java.lang.String,java.lang.String)'
            params:
              - '$.match'
              - '${jsonSerialize($.permissions)}'

  /security-settings/bulk:
    post:
      summary: Bulk upsert security settings
      operationId: bulkUpsertSecuritySettings
      tags: [AccessControl, Bulk]
      security: [ { oauth2_azure_ad: [broker.admin] } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/BulkRequest"
                - type: object
                  properties:
                    items:
                      type: array
                      items: { $ref: "#/components/schemas/SecuritySetting" }
      responses: { "207": { description: Multi-Status, content: { application/json: { schema: { $ref: "#/components/schemas/BulkResult" } } } } }
      x-raci: { allowed: [Admin] }

  /security-settings/{id}:
    get:
      summary: Get security setting
      operationId: getSecuritySetting
      tags: [AccessControl]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/SecuritySetting" } } } }
        "404": { description: Not Found }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'getRolesAsJSON(java.lang.String)'
            params: ['{id}']
    patch:
      summary: Update security setting
      operationId: updateSecuritySetting
      tags: [AccessControl]
      security: [ { oauth2_azure_ad: [broker.admin] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/IfMatch" } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/SecuritySetting" } } }
      responses:
        "200": { description: OK, headers: { ETag: { schema: { type: string } } }, content: { application/json: { schema: { $ref: "#/components/schemas/SecuritySetting" } } } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'removeSecuritySettings(java.lang.String)'
            params: ['{id}']
    delete:
      summary: Delete security setting
      operationId: deleteSecuritySetting
      tags: [AccessControl]
      security: [ { oauth2_azure_ad: [broker.admin] } ]
      parameters: [ { name: id, in: path, required: true, schema: { type: string } }, { $ref: "#/components/parameters/IfMatch" } ]
      responses: { "204": { description: No Content } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'removeSecuritySettings(java.lang.String)'
            params: ['{id}']

  # -------------------- Address Settings / Policies --------------------
  /address-settings:
    get:
      summary: List address settings
      operationId: listAddressSettings
      tags: [Policies]
      parameters: [ { $ref: "#/components/parameters/PageParam" }, { $ref: "#/components/parameters/PageSizeParam" } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { type: object, properties: { items: { type: array, items: { $ref: "#/components/schemas/AddressSettings" } }, page: { $ref: "#/components/schemas/PageMeta" } } } } } }
      x-raci: { allowed: [Admin, DataOwner] }
      x-impl:
        jolokia:
          read:
            - mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
              attribute: 'AddressSettingsAsJSON'
    post:
      summary: Create address setting
      operationId: createAddressSetting
      tags: [Policies]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      requestBody:
        required: true
        content: { application/json: { schema: { $ref: "#/components/schemas/AddressSettings" } } }
      responses: { "201": { description: Created, content: { application/json: { schema: { $ref: "#/components/schemas/AddressSettings" } } } } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'addAddressSettings(java.lang.String,java.lang.String)'
            params: ['$.match', '${jsonSerialize(body)}']

  /address-settings/bulk:
    post:
      summary: Bulk upsert address settings
      operationId: bulkUpsertAddressSettings
      tags: [Policies, Bulk]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/BulkRequest"
                - type: object
                  properties:
                    items:
                      type: array
                      items: { $ref: "#/components/schemas/AddressSettings" }
      responses: { "207": { description: Multi-Status, content: { application/json: { schema: { $ref: "#/components/schemas/BulkResult" } } } } }
      x-raci: { allowed: [Admin] }

  # -------------------- Runtime: Connections --------------------
  /connections:
    get:
      summary: List active connections
      operationId: listConnections
      tags: [Runtime]
      responses: { "200": { description: OK } }
      x-raci: { allowed: [Admin, DataOwner] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'listConnectionIDs()'

  /connections/{connectionId}/close:
    post:
      summary: Close a connection
      operationId: closeConnection
      tags: [Runtime]
      security: [ { oauth2_azure_ad: [broker.write] } ]
      parameters: [ { name: connectionId, in: path, required: true, schema: { type: string } } ]
      responses: { "202": { description: Accepted } }
      x-raci: { allowed: [Admin] }
      x-impl:
        jolokia:
          exec:
            mbean: 'org.apache.activemq.artemis:type=Broker,brokerName={brokerName}'
            operation: 'closeConnectionsForAddress(java.lang.String)'
            params: ['{connectionId}']

  # -------------------- Audit --------------------
  /audit:
    get:
      summary: List audit records
      operationId: listAudit
      tags: [Audit]
      parameters: [ { $ref: "#/components/parameters/PageParam" }, { $ref: "#/components/parameters/PageSizeParam" }, { $ref: "#/components/parameters/FilterParam" } ]
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  items: { type: array, items: { $ref: "#/components/schemas/AuditRecord" } }
      x-raci: { allowed: [Admin] }