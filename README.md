Why this aligns with Artemis & Jolokia

Same management surface, multiple transports. Artemis exposes the same management controls via JMX, Core, JMS, and through Jolokia (HTTP→JMX) which the web console also uses—so the API can reliably invoke Jolokia to perform server/queue/address/divert operations.
Queue/Address/Divert ops. Server-level operations like create/deploy/destroyQueue, createAddress/deleteAddress, and create/destroyDivert are available on the ActiveMQServerControl MBean (exact signatures vary by version—consult the Javadoc).
Permissions model. Access control in Artemis is per address match with permissions such as send, consume, manage, createDurableQueue, etc.—this maps cleanly to our /security-settings resources where you bind AD groups to those permissions.
Jolokia endpoint. Artemis logs the default Jolokia endpoint as …/console/jolokia and warns about security/CORS when exposing it—ensure the broker’s jolokia-access.xml is configured appropriately.


Operational guidance (non‑normative)

Token validation & groups claim. Use Azure AD client‑credentials; ensure your API gateway or auth layer includes the groups claim so RACI can map callers to AD groups.
Namespace ownership. Enforce namespaceScoped: true on write operations. For example, a caller in CN=MQ-DataOwners-finance,… can manage only resources where namespace=finance.
Bulk safety. All bulk endpoints return 207 Multi‑Status with per‑item results; use Idempotency-Key to make retried bulk creates safe.
ETags. Require If-Match on updates/deletes to avoid lost updates.
Jolokia security. If you front Jolokia with TLS offload, watch for mixed scheme behavior and fix CORS/Origin handling per Artemis/Jolokia guidance; the console docs call this out explicitly.
