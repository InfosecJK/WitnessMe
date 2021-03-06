# WitnessMe

<p align="center">
  <img src="https://user-images.githubusercontent.com/5151193/60783062-1f637c00-a106-11e9-83de-83ef88115f74.gif" alt="WitnessMe"/>
</p>

My take on a Web Inventory tool, heavily inspired by [Eyewitness](https://github.com/FortyNorthSecurity/EyeWitness). Takes screenshots of webpages using [Pyppeteer](https://github.com/miyakogi/pyppeteer) (headless Chrome/Chromium).

Supports Python 3.7+, fully asynchronous and has extra bells & whistles that make life easier.

## Why & what problems does this solve

- Python 3.7+
- No dependency/installation hell, works on a variety of *nix flavors
- AsyncIO provides Mad Max level speeds
- Headless chrome/chromium is the best.
- Provides a RESTful API.

## Sponsors
[<img src="https://www.blackhillsinfosec.com/wp-content/uploads/2016/03/BHIS-logo-L-300x300.png" width="130" height="130"/>](https://www.blackhillsinfosec.com/)
[<img src="https://handbook.volkis.com.au/assets/img/Volkis_Logo_Brandpack.svg" width="130" hspace="10"/>](https://volkis.com.au)
[<img src="https://user-images.githubusercontent.com/5151193/85817125-875e0880-b743-11ea-83e9-764cd55a29c5.png" width="200" vspace="21"/>](https://qomplx.com/blog/cyber/)
[<img src="https://user-images.githubusercontent.com/5151193/86521020-9f0f4e00-be21-11ea-9256-836bc28e9d14.png" width="250" hspace="20"/>](https://ledgerops.com)

## Table of Contents

Note, the documentation is still a WIP. I've released this early because of CVE-2020-5902.

- [WitnessMe](#witnessme)
  * [Quick starts](#quick-starts)
    + [Finding F5 Load Balancers Vulnerable to CVE-2020-5902](#finding-f5-load-balancers-vulerable-to-cve-2020-5902)

## Quick Starts
### Finding F5 Load Balancers Vulnerable to CVE-2020-5902

**Note it is highly recommended to give the Docker container at least 4GB of RAM during large scans as Chromium can be a resource hog. If you keep running into "Page Crash" errors, it's because your container does not have enough memory. On Mac/Windows you can change this by clicking the Docker Task Bar Icon -> Preferences -> Resources. For Linux, refer to Docker's documentation**

Install WitnessMe using Docker:

```console
docker pull byt3bl33d3r/witnessme
```

Get the `$IMAGE_ID` from the `docker images` command output, then run the following command to drop into a shell inside the container. Additionally, specify the `-v` flag to mount the current directory inside the container at the path `/transfer` in order to copy the scan results back to your host machine (if so desired):

```console
docker run -it --entrypoint=/bin/sh -v $(pwd):/transfer $IMAGE_ID
```

Scan your network using WitnessMe, it can accept multiple .Nessus files, Nmap XMLs, IP ranges/CIDRs. Example:

```console
witnessme 10.0.1.0/24 192.168.0.1-20 ~/my_nessus_scan.nessus ~/my_nmap_scan.nessus.
```

After the scan is finished, a folder will have been created in the current directory with the results. Access the results using the `wmdb` command line utility:

```console
wmdb scan_2020_$TIME/
```

To quickly identify F5 load balancers, first perform a signature scan using the `scan` command. Then search for "BIG-IP" or "F5" using the `servers` command (this will search for the "BIG-IP" and "F5" string in the signature name, page title and server header):

![image](https://user-images.githubusercontent.com/5151193/86619581-43fc6900-bf91-11ea-9a01-ba8ce09c3f3b.png)

Additionally, you can generate an HTML or CSV report using the following commands:
```console
WMDB ≫ generate_report html
```
```console
WMDB ≫ generate_report csv
```

You can then copy the entire scan folder which will contain all of the reports and results to your host machine by copying it to the `/transfer` folder.


## Installation

### Docker

Running WitnessMe from a Docker container is fully supported and is the recommended way of using the tool.

Pull the image from Docker Hub:

```bash
docker pull byt3bl33d3r/witnessme
```

You can then spin up a docker container, run it like the main `witnessme.py` script and pass it the same arguments:

```bash
docker --rm -ti $IMAGE_ID https://google.com 192.168.0.1/24
```

## Deploying to the Cloud (™)

Since WitnessMe has a RESTful API now, you can deploy it to the magical cloud and perform scanning from there. This would have a number of benefits, including giving you a fresh external IP on every scan (More OPSEC safe when assessing attack surface on Red Teams).

There are a number of ways of doing this, you can obviously do it the traditional way (e.g. spin up a machine, install docker etc..).

Recently cloud service providers started offering ways of running Docker containers directly in a fully managed environment. Think of it as serverless functions (e.g. AWS Lambdas) only with Docker containers.

This would technically allow you to really quickly deploy and run WitnessMe (or really anything in a Docker container) without having to worry about underlying infrastructure and removes a lot of the security concerns that come with that.

Below are some of the ones I've tried along with the steps necessary to get it going and any issues I encountered.

### GCP Cloud Run

Cloud Run is by far the easiest of these services to work with.

This repository includes the `cloudbuild.yaml` file necessary to get this setup and running.

**Unfortunately, it seems like Cloud Run doesn't allow outbound internet access to containers, if anybody knows of a way to get around this please get in touch**

From the repositories root folder (after you authenticated and setup a project), these two commands will automatically build the Docker image, publish it to the Gcloud Container Registry and deploy a working container to Cloud Run:

```bash
gcloud builds submit --config cloudbuild.yaml
gcloud run deploy --image gcr.io/$PROJECT_ID/witnessme --platform managed
```

The output will give you a HTTPS url to invoke the WitnessMe RESTful API from :)

When you're done:

```bash
gcloud run services delete witnessme
gcloud container images delete gcr.io/$PROJECT_ID/witnessme
```

### AWS ECS/Fargate

TO DO

## Usage & Examples

There are 3 main scripts:

- `witnessme.py`: is the main CLI interface.
- `wmdb.py`: allows you to browse the database (created on each scan) to view results.
- `wmapi.py`: provides a RESTful API to schedule, start, stop and monitor scans.

```
usage: witnessme.py [-h] [-p PORTS [PORTS ...]] [--threads THREADS]
                    [--timeout TIMEOUT]
                    target [target ...]

positional arguments:
  target                The target IP(s), range(s), CIDR(s) or hostname(s)

optional arguments:
  -h, --help            show this help message and exit
  -p PORTS [PORTS ...], --ports PORTS [PORTS ...]
                        Ports to scan if IP Range/CIDR is provided (default:
                        [80, 8080, 443, 8443])
  --threads THREADS     Number of concurrent threads (default: 20)
  --timeout TIMEOUT     Timeout for each connection attempt in seconds
                        (default: 15)
```

Can accept a mix of .Nessus file(s), Nmap XML file(s), files containing URLs and/or IPs, IP addresses/ranges/CIDRs and URLs. Long story short, should be able to handle anything you throw at it:

```bash
python witnessme.py 192.168.1.0/24 192.168.1.10-20 https://bing.com ~/my_nessus_scan.nessus ~/my_nmap_scan.xml ~/myfilewithURLSandIPs
```

*Note: as of writing, WitnessMe detects .Nessus and NMap files by their extension so make sure Nessus files have a `.nessus` extension and NMap scans have a `.xml` extension*

If an IP address/range/CIDR is specified as a target, WitnessMe will attempt to screenshot HTTP & HTTPS pages on ports 80, 8080, 443, 8443 by default. This is customizable with the `--port` argument.

Once a scan is completed, a folder with all the screenshots and a database will be in the current directory, point `wmdb.py` to the database in order to see the results.

```bash
python wmdb.py scan_2019_11_05_021237/witnessme.db
```

Pressing tab will show you the available commands and a help menu:

<p align="center">
  <img width="534" src="https://user-images.githubusercontent.com/5151193/68552696-11d14200-03d7-11ea-828f-3c744e58df86.png" alt="ScreenPreview"/>
</p>

## Searching the Database

The `servers` and `hosts` commands in the `wmdb.py` CLI accept 1 argument. WMCLI is smart enough to know what you're trying to do with that argument

### Server Command

No arguments will show all discovered servers. Passing it an argument will search the `title` and `server` columns for that pattern (it's case insensitive).

For example if you wanted to search for all discovered Apache Tomcat servers:
- `servers tomcat` or `servers 'apache tomcat'`

Similarly if you wanted to find servers with a 'login' in the title:
- `servers login`

### Hosts Command

No arguments will show all discovered hosts. Passing it an argument will search the `IP` and `Hostname` columns for that pattern (it's case insensitive). If the value corresponds to a Host ID it will show you the host information and all of the servers discovered on that host which is extremely useful for reporting purposes and/or when targeting specific hosts.

### Signature Scan

You can perform a signature scan on all discovered services using the `scan` command.

## Call for Signatures!

If you run into a new webapp write a signature for it! It's beyond simple and they're all in YAML!

Don't believe me? Here's the AirOS signature (you can find them all in the [signatures directory](https://github.com/byt3bl33d3r/WitnessMe/tree/master/witnessme/signatures)):

```yaml
credentials:
- password: ubnt
  username: ubnt
name: AirOS
signatures:
- airos_logo.png
- form enctype="multipart/form-data" id="loginform" method="post"
- align="center" class="loginsubtable"
- function onLangChange()
# AirOS ubnt/ubnt
```

Yup that's it. Just plop it in the signatures folder and POW! Done.

# Preview Screenshots Directly in the Terminal (ITerm2 on MacOSX)

If you're using ITerm2 on MacOSX, you can preview screenshots directly in the terminal using the `show` command:

<p align="center">
  <img src="https://user-images.githubusercontent.com/5151193/68194496-5e012a00-ff72-11e9-9ccd-6a50aa384f3e.png" alt="ScreenPreview"/>
</p>

## To Do

1. ~~Store server info to a database~~
2. HTML report generation
3. ~~Command line script to search database~~
4. ~~Support NMap & .nessus files as input~~
5. ~~Web server categorization~~
6. Signature scanning
6. ~~Accept URLs as targets (stdin, files)~~
7. Add support for previewing screenshots in *nix terminals using w3m
