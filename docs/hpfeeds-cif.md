hpfeeds-cif
=============
The hpfeeds-cif container, when added to a CHN-Server instance, will forward 
an "observation" of the malicious activity to a specified Collective 
Intelligence Framework (CIF) instance. You can read more about CIF [here](https://csirtgadgets.com/collective-intelligence-framework/) 


# Adding hpfeeds-cif to CHN-Server
The simplest way to integrate CHN reporting to CIF is to:

First, include this stanza in the docker-compose.yml file for CHN-server:
```dockerfile
  hpfeeds-cif:
    image: stingar/hpfeeds-cif:1.7
    privileged: true
    volumes:
      - ./hpfeeds-cif.sysconfig:/etc/default/hpfeeds-cif
    links:
      - hpfeeds:hpfeeds
      - mongodb:mongodb
```
Next, add the following hpfeeds-cif.sysconfig configuration file:
```bash
HPFEEDS_HOST='hpfeeds'
HPFEEDS_PORT=10000
IDENT=hpfeeds-cif-${RANDOM}

MONGODB_HOST='mongodb'
MONGODB_PORT=27017

CIF_HOST='https://YOUR.CIF.SERVER'
CIF_TOKEN='YOUR_API_TOKEN_WITH_WRITE_ACCESS'
CIF_PROVIDER='org-chn'
CIF_TLP='green'
CIF_CONFIDENCE='8'
CIF_TAGS='honeypot'
CIF_GROUP='everyone'
# Set the below value to True if your CIF instance uses a valid, CA-signed, certificate
CIF_VERIFY_SSL=False

# Set to False if you wish to submit private addresses to CIF
IGNORE_RFC1918=True

# Include the honeypot specific tags in CIFv3
INCLUDE_HP_TAGS=False
```

Once the docker-compose.yml is updated and the hpfeeds-cif.sysconfig file is 
present, you can simply:

```bash
docker-compose down && docker-compose up -d
```
To examine logs of the transactions with the CIF instance, run:

```bash
docker-compose logs hpfeeds-cif
```

As troubleshooting a remote CIF instance can prove difficult, a high level of
 debugging logs are turned on by default. __*Warning*: The host and API key 
 will be present in the container logs!__
 
 There is currently no option for turning this logging off; if this presents 
 an issue for you, please file an issue with the [repo](https://github.com/CommunityHoneyNetwork/hpfeeds-cif/issues)

# Pulling data from CIF
##Installing the CIF client
One of the easiest ways to pull data from a CIF instance is to use the [CIF 
client](https://github.com/csirtgadgets/bearded-avenger-sdk-py/wiki) from the
 CIF project. Installing a CIFv3 client is as easy as `pip install 'cifsdk>=3.0.0,<4.0'`. Once the client is installed, you should save your credentials 
 in a configuration file, where the format is:
 
```yaml
token: your_cif_token_with_read_rights_here
remote: https://remote.fqdn.cif.server.here
no_verify_ssl: false
```

__*Note:*__ The 'no_verify_ssl' option sets whether or not to do strict TLS 
checking on the certificate presented when connecting. When you have a 
properly validated certificate installed on your CIF instance, set this value
 to 'false'; if your CIF instance uses a self-signed certificate, use 'true' 
 here.
 
## Selecting and formatting data
 Once you have the CIF client installed, you can use the client to [select 
 and pull data from the CIF instance in a variety of formats](https://github.com/csirtgadgets/bearded-avenger-deploymentkit/wiki/Where-do-I-start-Feeds).
 
There are many options in the CIF client to select and format the data you'll 
receive. We suggest you explore [this link](https://github.com/csirtgadgets/bearded-avenger-deploymentkit/wiki/Where-do-I-start-Feeds)
to understand more of the options available. 

A basic useful query would be:

```bash
$ cif --tags honeypot --itype ipv4 --last-hour
+-------+----------+----------------------------+-----------------+----------------------------+----------------------------+-------+------------------+-------------+------------+-------+----------------+
|  tlp  |  group   |         reporttime         |    indicator    |         firsttime          |          lasttime          | count |       tags       | description | confidence | rdata | provider       |
+-------+----------+----------------------------+-----------------+----------------------------+----------------------------+-------+------------------+-------------+------------+-------+----------------+
| green | everyone | 2019-02-15T23:15:34.72083Z |   0.0.77.37     | 2019-02-15T23:15:33.52882Z | 2019-02-15T23:15:33.64677Z |   2   | honeypot,dionaea |     None    |    8.0     |  None | cnh-sandbox |
| green | everyone | 2019-02-15T23:16:45.13496Z |  0.0.230.237    | 2019-02-15T23:12:24.33256Z | 2019-02-15T23:16:42.05893Z |   3   | honeypot,dionaea |     None    |    8.0     |  None | cnh-sandbox |
``` 

The CIF client also offers options for formatting the data into csv, json, as
 well as application specific formats such as [Zeek Intel Framework](https://github.com/csirtgadgets/bearded-avenger-deploymentkit/wiki/Where-do-I-start-with-Integrations)

CSV, selecting fields of interest:
 
```bash
 $ cif --tags honeypot --itype ipv4 --last-hour -f csv --columns indicator,lasttime,count,tags,asn
"indicator","lasttime","count","tags","asn"
"0.0.133.212","2019-02-16T00:14:14.407959Z","10","honeypot,dionaea","12389.0"
"0.0.161.58","2019-02-16T00:47:42.705318Z","5","honeypot,dionaea","39130.0"
```

JSON plus some jq magic:

```bash
$ cif --tags honeypot --itype ipv4 --last-hour -f json --columns indicator,lasttime,count,tags,asn | jq
[
  {
    "count": 10,
    "indicator": "0.0.133.212",
    "lasttime": "2019-02-16T00:14:14.407959Z",
    "asn": 12389,
    "tags": "honeypot,dionaea"
  },
  {
    "count": 5,
    "indicator": "0.0.161.58",
    "lasttime": "2019-02-16T00:47:42.705318Z",
    "asn": 39130,
    "tags": "honeypot,dionaea"
  }
]
```

Many protection devices want a list of IP addresses in a file, one per line. 
This sort of format is easy to achieve with the CIF client and some light 
command line magic.

```bash
$ cif --tags honeypot --itype ipv4 --last-hour -f csv --columns indicator | tail -n +2 | sed -e 's/"//g' > file
$ cat file
0.0.161.58
0.0.160.198
0.0.25.156
```

## Rolling your own solution
In addition to using the CIF command line interface, you can also use the SDK
 as the basis for your own solution. If you prefer to work in a language 
 other than python, you can program directly against the [API](https://github.com/csirtgadgets/bearded-avenger-deploymentkit/wiki/REST-API).
