# Getting Started with Singer

Singer is an open source standard for moving data between databases,
web APIs, files, queues, and just about anything else you can think
of. The [Singer spec] describes how data extraction scripts — called
“Taps” — and data loading scripts — called “Targets” — should
communicate using a standard JSON-based data format over `stdout`. By
conforming to this spec, Taps and Targets can be used in any
combination to move data from any source to any destination.

**Topics**

 - [Using Singer to populate Google Sheets](#using-singer-to-populate-google-sheets)
 - [Developing a Tap](#developing-a-tap)
 - [Additional Resources](#additional-resources)
 
## Using Singer to populate Google Sheets

The [Google Sheets Target] can be combined with any Singer Tap to
populate a Google Sheet with data. This example will use currency
exchange rate data from the [Fixer.io Tap]. [Fixer.io] is a free API for
current and historical foreign exchange rates published by the
European Central Bank.

The steps are:
 1. [Activate the Google Sheets API](#step-1---activate-the-google-sheets-api)
 1. [Configure the Target](#step-2---configure-the-target)
 1. [Install](#step-3---install)
 1. [Run](#step-4---run)
 1. [Save State (optional)](#step-5---save-state-optional)

### Step 1 - Activate the Google Sheets API

 (originally found in the [Google API
 docs](https://developers.google.com/sheets/api/quickstart/python))
 
 1. Use [this
 wizard](https://console.developers.google.com/start/api?id=sheets.googleapis.com)
 to create or select a project in the Google Developers Console and
 activate the Sheets API. Click Continue, then Go to credentials.

 1. On the **Add credentials to your project** page, click the
 **Cancel** button.

 1. At the top of the page, select the **OAuth consent screen**
 tab. Select an **Email address**, enter a **Product name** if not
 already set, and click the **Save** button.

 1. Select the **Credentials** tab, click the **Create credentials**
 button and select **OAuth client ID**.

 1. Select the application type **Other**, enter the name "Singer
 Sheets Target", and click the **Create** button.

 1. Click **OK** to dismiss the resulting dialog.

 1. Click the Download button to the right of the client ID.

 1. Move this file to your working directory and rename it
 `client_secret.json`.

### Step 2 - Configure the Target

Created a file called `config.json` in your working directory,
following [config.sample.json](https://github.com/singer-io/target-gsheet/blob/master/config.sample.json). The required
`spreadsheet_id` parameter is the value between the "/d/" and the
"/edit" in the URL of your spreadsheet. For example, consider the
following URL that references a Google Sheets spreadsheet:

```
https://docs.google.com/spreadsheets/d/1qpyC0XzvTcKT6EISywvqESX3A0MwQoFDE8p-Bll4hps/edit#gid=0
```

The ID of this spreadsheet is
`1qpyC0XzvTcKT6EISywvqESX3A0MwQoFDE8p-Bll4hps`.


### Step 3 - Install

First, make sure Python 3 is installed on your system or follow these
installation instructions for [Mac](python-mac) or
[Ubuntu](python-ubuntu).

`target-gsheet` can be run with any [Singer Tap] to move data from
sources like [Braintree], [Freshdesk] and [Hubspot] to Google
Sheets. We'll use the [Fixer.io Tap] - which pulls currency exchange
rate data from a public data set - as an example.

These commands will install `tap-fixerio` and `target-gsheet` with
pip:

```bash
› pip install target-gsheet tap-fixerio
```

### Step 4 - Run

This command will pipe the output of `tap-fixerio` to `target-gsheet`,
using the configuration file created in Step 2:

```bash
› tap-fixerio | target-gsheet -c config.json
    INFO Replicating the latest exchange rate data from fixer.io
    INFO Tap exiting normally
```

`target-gsheet` will attempt to open a new window or tab in your
default browser to perform authentication. If this fails, copy the URL
from the console and manually open it in your browser.

If you are not already logged into your Google account, you will be
prompted to log in. If you are logged into multiple Google accounts,
you will be asked to select one account to use for the
authorization. Click the **Accept** button to allow `target-gsheet` to
access your Google Sheet.  You can close the tab after the signup flow
is complete.

Each stream generated by the Tap will be written to a different sheet
in your Google Sheet. For the [Fixer.io Tap] you'll see a single sheet
named `exchange_rate`.

### Step 5 - Save State (optional)

When `target-gsheet` is run as above it writes log lines to `stderr`,
but `stdout` is reserved for outputting **State** messages. A State
message is a JSON-formatted line with data that the Tap wants
persisted between runs - often "high water mark" information that the
Tap can use to pick up where it left off on the next run. Read more
about State messages in the [Singer spec].

Targets write State messages to `stdout` once all data that appeared
in the stream before the State message has been processed by the
Target. Note that although the State message is sent into the target,
in most cases the target's process won't actually store it anywhere or
do anything with it other than repeat it back to `stdout`.

Taps like the [Fixer.io Tap] can also accept a `--state` argument
that, if present, points to a file containing the last persisted State
value.  This enables Taps to work incrementally - the State
checkpoints the last value that was handled by the Target, and the
next time the Tap is run it should pick up from that point.

To run the [Fixer.io Tap] incrementally, point it to a State file and
capture the persister's `stdout` like this:

```bash
› tap-fixerio --state state.json | target-gsheet -c config.json >> state.json
› tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
(rinse and repeat)
```

## Developing a Tap

If you can't find an existing Tap for your data source, then it's time
to build your own.

**Topics**:
 - [Hello, world](#hello-world)
 - [A Python Tap](#a-python-tap)
 
### Hello, world

A Tap is just a program, written in any language, that outputs data to
`stdout` according to the [Singer spec]. In fact, your first Tap can
be written from the command line, without any programming at all:

```bash
› printf '{"type":"RECORD","stream":"hello","record":{"value":"world"}}\n'
```

This writes the datapoint `{"value":"world"}` to the *hello*
stream. That data can be piped into any Target, like the [Google Sheets
Target], over `stdin`:

```bash
› printf '{"type":"RECORD","stream":"hello","record":{"value":"world"}}\n' | target-gsheet -c config.json
```

### A Python Tap

To move beyond *Hello, world* you'll need a real programming language.
Although any language will do, we have built a Python library to help
you get up and running quickly.

Let's write a Tap called `tap_ip.py` that retrieves the current
 IP using icanhazip.com, and writes that data with a timestamp.

First, install the [Singer helper library] with `pip`:

```bash
› pip install singer-python
```

Then, open up a new file called `tap_ip.py` in your favorite editor.

```python
import singer
import urllib.request
from datetime import datetime, timezone
```

We'll use the `datetime` module to get the current timestamp, the
`singer` module to write data to `stdout` in the correct format, and
the `urllib.request` module to make a request to icanhazip.com.

```python
now = datetime.now(timezone.utc).isoformat()
schema = {'properties':	
	  {'ip': {'type': 'string'},
           'timestamp': {'type': 'string',
           	         'format': 'date-time'}}}
```

This sets up some of the data we'll need - the current time, and the
schema of the data we'll be writing to the stream formatted as a [JSON
Schema].

```python
with urllib.request.urlopen('http://icanhazip.com') as response:
    ip = response.read().decode('utf-8').strip()
    singer.write_schema('my_ip', schema, 'timestamp')
    singer.write_records('my_ip', [{'timestamp': now,'ip': ip}])
```

Finally, we make the HTTP request, parse the response, and then make
two calls to the `singer` library:

 - `singer.write_schema` which writes the schema of the `my_ip` stream and defines its primary key
 - `singer.write_records` to write a record to that stream

We can send this data to Google Sheets by running our new Tap
with the [Google Sheets Target]:

```bash
› python tap_ip.py | target-gsheet -c config.json
```

## Additional Resources

Join the [Singer Slack channel] to get help from members of the Singer
community.

---

Copyright &copy; 2017 Stitch

[Singer spec]: SPEC.md
[Singer Tap]: https://singer.io
[Braintree]: https://github.com/singer-io/tap-braintree
[Freshdesk]: https://github.com/singer-io/tap-freshdesk
[Hubspot]: https://github.com/singer-io/tap-hubspot
[Fixer.io Tap]: https://github.com/singer-io/tap-fixerio
[Fixer.io]: http://fixer.io
[python-mac]: http://docs.python-guide.org/en/latest/starting/install3/osx/
[python-ubuntu]: https://www.digitalocean.com/community/tutorials/how-to-install-python-3-and-set-up-a-local-programming-environment-on-ubuntu-16-04
[Google Sheets Target]: https://github.com/singer-io/target-gsheet
[Singer helper library]: https://github.com/singer-io/singer-python
[JSON Schema]: http://json-schema.org/
[Singer Slack channel]: https://singer-slackin.herokuapp.com/

