# Patroneos

Patroneos provides a layer of protection for EOSIO nodes designed to protect against some of the basic Denial of Service attack vectors.
It operates in two modes, filter and log. When running in filter mode, it proxies requests to the node through a series of middleware that perform various checks and logs success or failures to the log agents. When running in log mode, it receives logs from the filter agent and writes them to a file in a format that fail2ban can process.

## Configuration
The configuration can be update through the `config.json` file or by sending a `POST` request with the new configuration in the request body to `/config`.

## Middleware Verification Layer

* validateJSON
    * This middleware checks that the body provided can be parsed into a JSON object.

* validateSignatures
    * This middleware checks that the number of signatures on the transaction are not greater than the defined maximum.

* validateContract
    * This middleware checks that the contract is not in a list of blacklisted contracts.

* validateTransactionSize
    * This middleware checks that the size of the transaction data does not exceed the defined maximum.

## Proof of Concept

We have provided a docker-compose.yml that will allow you to stand up a proof of concept (POC) environment of Patroneos. However, this only works with Docker for Linux. Both Docker for Mac and Docker for Windows have incompatible network stacks, that prevent proper functionality of Patroneos (On both Mac and Windows, the source IP address for all requests appear to come from the same IP address. This causes a ban to block all traffic to nodeos, instead of blocking just the offending client).

We have provided sample configurations for HAProxy, fail2ban and Patroneos (both in filter mode and log mode). These can be used to stand up Patroneos in whichever type of environment (Bare Metal, Virtualized, Containerized, etc.) you choose. The configurations can be found within the docker folder.

### Infrastructure

Our POC environment is made up of three layers. The proxy, filter and nodeos

- Proxy: HAProxy 1.8.8, fail2ban 0.9.6, and Patroneos operating in log mode.
- Filter: Patroneos operating in filter mode.
- Nodeos: We are using the latest docker image of eosio/eos.

#### Traffic Flow

*End User* -> Proxy -> Filter -> **Nodeos** -> Filter -> Proxy -> *End User*

#### Proxy

For this layer, we are proxying requests through HAProxy. At this point, we terminate SSL, and then route the request to the filter layer.

The proxy runs Patroneos in log mode (listens for rule violations from the filter(s), and logs them to a log file.

Finally, the proxy runs fail2ban which watches the Patroneos log file for rule violations. As the log file is populated, fail2ban looks for specific patterns that categorize the type of rule violation, and when a threshold is met, it issues an IP ban via iptables for a specified length of time preventing further requests from being received.

#### Filter

Patroneos (in filter mode) inspects the requests for multiple rule violations. If a violation is found, it immediately rejects the request. If not, it forwards the request to nodeos.

If a rule violation is detected, the request is immediately rejected. Additionally, Patroneos broadcasts the rule violation to all the proxies running Patroneos in log mode, so that they can log the violation.

#### Nodeos

Once the request is inspected and found to not violate any rules, the request is then routed to Nodeos to be handled as it normally would.

### Redundancy and Auto Scaling

Our POC environment only contains one instance of proxy, filter, and nodeos. For a production environment, you will likely require redundancy. Due to the large number environments Patroneos may be ran within, we have not baked in a solution for network autodiscovery. Instead, we have created an endpoint (/config) within Patroneos that can be used to update the configuration of Patroneos without restarting the daemon. From here, you could use a tool such as Ansible/Puppet/Chef/etc. to fire up a new instance of the filter, and then do `POST` requests to all the proxies to update the configuration with the new filter that was added.

## Documentation for Third Party Utilities

- [HAProxy](http://www.haproxy.org/#docs)
- [fail2ban](https://fail2ban.readthedocs.io/en/latest/)
