# Secure Linux Server on Google Cloud

A hands-on cloud security project demonstrating how to deploy a Linux
web server on Google Cloud, restrict network access with VPC firewall
rules, validate the exposed ports with Nmap, and review allowed traffic
in Cloud Logging.

**Tools:** Google Cloud Compute Engine, VPC Firewall, Cloud Logging,
Debian Linux, Nginx, Nmap, Kali Linux

------------------------------------------------------------------------

## Goal

-   Host a public web page over HTTP (TCP port 80).
-   Restrict SSH (TCP port 22) to an authorized source IP and Google's
    browser-based SSH range.
-   Test the server's exposed ports from two different networks using
    Nmap.
-   Review allowed firewall traffic in Google Cloud Logs Explorer.

## Environment

  Item               Configuration
  ------------------ --------------------------------
  Cloud platform     Google Cloud Compute Engine
  VM machine type    e2-micro
  Operating system   Debian 13
  Boot disk          10 GB standard persistent disk
  Web server         Nginx
  Scanner            Nmap 7.99 on Kali Linux

*Project identifiers, public IP addresses, and other account-specific
values are intentionally omitted.*

## Firewall configuration

  -------------------------------------------------------------------------------------------------------------
  Rule                       Direction   Source                        Protocol /  Purpose          Logging
                                                                       port                         
  -------------------------- ----------- ----------------------------- ----------- ---------------- -----------
  `allow-ssh-myip`           Ingress     `<AUTHORIZED_PUBLIC_IP>/32`   TCP 22      Restricted       Enabled
                                         and Google's browser-SSH                  administration   
                                         range                                     access           

  `allow-http`               Ingress     `0.0.0.0/0`                   TCP 80      Public web       Enabled
                                                                                   access           

  `default-allow-internal`   Ingress     Internal VPC range            All         Internal VPC     Not enabled
                                                                                   communication    
  -------------------------------------------------------------------------------------------------------------

The default SSH, RDP, and ICMP firewall rules were removed to avoid
leaving those services broadly exposed.

## Implementation

1.  Created a Google Cloud project and configured a budget alert.

2.  Deployed a Debian Linux VM on Compute Engine.

3.  Configured the ingress firewall rules listed above.

4.  Installed Nginx and published a simple test page:

    ``` bash
    sudo apt update
    sudo apt install nginx -y
    echo "<h1>Secure GCP Server</h1>" | sudo tee /var/www/html/index.html
    ```

5.  Tested the server from Kali Linux using Nmap.

6.  Reviewed firewall activity in Google Cloud Logs Explorer.

## Nmap verification

The server was scanned from two different network locations. The actual
server address is replaced with a placeholder in the commands below.

**Scan from the network permitted by the SSH rule:**

``` bash
nmap -Pn -T4 --max-retries 1 --top-ports 1000 <SERVER_PUBLIC_IP>
```

Observed results:

-   `22/tcp` --- open (SSH)
-   `80/tcp` --- open (HTTP)
-   998 other scanned TCP ports --- filtered (no response)

**Scan from a different network:**

``` bash
nmap -Pn -p 22,80 <SERVER_PUBLIC_IP>
```

Observed results:

-   `22/tcp` --- filtered
-   `80/tcp` --- open (HTTP)

**What this demonstrates:** HTTP remained publicly reachable, while SSH
was reachable from the authorized network and filtered when tested from
another network.

## Firewall logging

Firewall logging was enabled on the relevant rules. The following Logs
Explorer query was used to find firewall log entries:

``` text
log_id("compute.googleapis.com/firewall")
```

The inspected log entries showed connection details, the `ALLOWED`
disposition, and the firewall rule that matched the traffic.

The default-deny blocked traffic did not appear as an equivalent
firewall log entry in this test. The Nmap scan from the second network
provided the evidence that SSH was filtered there.

## Screenshots

Place the evidence images in a folder named `screenshots/`.

  -----------------------------------------------------------------------
  Screenshot                          Evidence
  ----------------------------------- -----------------------------------
  `web-page.png`                      Nginx test page

  `firewall-rules.png`                Configured VPC firewall rules

  `nmap-normal-networking.png`        Nmap results from the authorized
                                      network

  `nmap-other-network.png`            Nmap results from a different
                                      network

  `firewall-log-details.png`          Example firewall log details

  `firewall-log-rule-details.png`     Firewall rule and source-range
                                      details

  `logs-explorer-query.png`           Logs Explorer query and results
  -----------------------------------------------------------------------

Before publishing screenshots, redact public IP addresses, project IDs,
account identifiers, and hostnames that could identify your cloud
resources. Keep the rule names, protocols, ports, scan states, and log
disposition visible.

## Key learnings

-   Firewall rules should expose only the services a server needs.
-   Administrative access should be restricted by source address.
-   Nmap can help verify whether firewall rules behave as intended.
-   Testing from different networks can reveal differences in access.
-   Cloud firewall logs help investigate allowed connections and
    identify the rule that permitted them.

## Cleanup

Delete the test VM and any lab resources that are no longer needed.
Review the project's billing page to check for any remaining billable
resources.

## Note

This is a learning project. The VM hosted a simple static test page and
was used for controlled security testing.
