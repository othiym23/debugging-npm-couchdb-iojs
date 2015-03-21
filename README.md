#### taken from [npm/npm#7699](https://github.com/npm/npm/issues/7699)

New nodes have a new (which is to say working) HTTP keep-alive agent. So far it seems to cause no problems for npm in practice, but the npm tests (including in continuous integration) consistently hang when running the tests that hit a local CouchDB instance to test the CouchDB application that powers npm's registry.

### steps to reproduce:

1. ensure that you have a recent version of io.js installed
2. ensure that you have CouchDB installed
3. clone npm/npm-registry-couchapp and run `npm install`
4. run `npm test`

### what should happen

The tests should proceed to completion without issue in ~30 seconds. This is what will happen if you swap out io.js or Node.js 0.12 with Node.js 0.10.x.

### what actually happens

The tests hang about 2/3 of the way through the first steps, which are for account creation. Eventually it times out. npm is left hanging partway through an HTTP request immediately after getting `ECONNRESET` or `EPIPE` from the CouchDB end. A potentially salient detail is that a number of requests are issued in rapid-fire succession, and that some of the responses have 40x codes because they're authentication failures or "document (user) not found" responses.

### debugging so far

* Even turned up to `debug`, the CouchDB logs for the tests reveal no interesting or unusual information.
* Wireshark captures of the transactions show some strange behavior with RSTs and duplicate ACKs, but not in a way that reveals why those RSTs are happening.
* running the tests with `NODE_DEBUG=http npm test` basically backs up the information revealed by Wireshark – the connections are getting closed on the CouchDB end, more or less midstream, and the subsequent retries are hanging.

Transcript:

```
% node --version
v1.6.1
% couchdb -V
couchdb - Apache CouchDB 1.6.1

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
% git clone npm/npm-registry-couchapp
Cloning into 'npm-registry-couchapp'...
done.
% cd npm-registry-couchapp
% npm install
npm WARN package.json Dependency 'json' exists in both dependencies and devDependencies, using 'json@^9.0.2' from dependencies
npm WARN engine follow@0.11.4: wanted: {"node":"0.10.x || 0.8.x"} (current: {"node":"1.6.1","npm":"2.7.4"})
which@1.0.9 node_modules/which

rimraf@2.2.8 node_modules/rimraf

semver@4.3.1 node_modules/semver

parse-json-response@1.0.1 node_modules/parse-json-response
└── once@1.3.1 (wrappy@1.0.1)

json@9.0.3 node_modules/json

request@2.53.0 node_modules/request
├── caseless@0.9.0
├── json-stringify-safe@5.0.0
├── aws-sign2@0.5.0
├── forever-agent@0.5.2
├── stringstream@0.0.4
├── tunnel-agent@0.4.0
├── oauth-sign@0.6.0
├── isstream@0.1.2
├── node-uuid@1.4.3
├── qs@2.3.3
├── combined-stream@0.0.7 (delayed-stream@0.0.5)
├── form-data@0.2.0 (async@0.9.0)
├── mime-types@2.0.10 (mime-db@1.8.0)
├── http-signature@0.10.1 (assert-plus@0.1.5, asn1@0.1.11, ctype@0.5.3)
├── tough-cookie@0.12.1 (punycode@1.3.2)
├── bl@0.9.4 (readable-stream@1.0.33)
└── hawk@2.3.1 (cryptiles@2.0.4, sntp@1.0.9, boom@2.6.1, hoek@2.11.1)

tap@0.7.1 node_modules/tap
├── yamlish@0.0.6
├── inherits@2.0.1
├── buffer-equal@0.0.1
├── slide@1.1.6
├── deep-equal@1.0.0
├── nopt@3.0.1 (abbrev@1.0.5)
├── mkdirp@0.5.0 (minimist@0.0.8)
├── difflet@0.2.6 (deep-is@0.1.3, charm@0.1.2, traverse@0.6.6)
├── glob@4.5.3 (once@1.3.1, inflight@1.0.4, minimatch@2.0.4)
└── runforcover@0.0.2 (bunker@0.1.2)

couchapp@0.11.0 node_modules/couchapp
├── watch@0.8.0
├── url@0.10.3 (punycode@1.3.2, querystring@0.2.0)
├── coffee-script@1.9.1
├── connect@3.3.5 (utils-merge@1.0.0, parseurl@1.3.0, debug@2.1.3, finalhandler@0.3.4)
├── nano@6.1.2 (errs@0.3.2, underscore@1.8.2, debug@2.1.3, follow@0.11.4)
└── http-proxy@0.8.7 (colors@0.6.2, pkginfo@0.2.3, optimist@0.3.7)
bauchelain% npm test

> npm-registry-couchapp@2.6.7 test /Users/ogd/Documents/projects/npm-registry-couchapp
> tap test/*.js

ok test/00-setup.js ..................................... 8/8
not ok test/01-adduser.js ............................... 5/6
    Command: "/usr/local/bin/iojs 01-adduser.js"
    TAP version 13
    ok 1 (unnamed assert)
    ok 2 (unnamed assert)
    ok 3 (unnamed assert)
    ok 4 (unnamed assert)
    ok 5 (unnamed assert)
    not ok 6 test/01-adduser.js
      ---
        exit:     ~
        timedOut: true
        signal:   SIGTERM
        command:  "/usr/local/bin/iojs 01-adduser.js"
      ...

    1..6
    # tests 6
    # pass  5
    # fail  1

ok test/01b-user-views.js ............................... 5/5
ok test/01c-user-updates.js ............................. 4/4
ok test/01d-whoami.js ................................... 4/4
ok test/02-publish.js ................................. 35/35
ok test/02b-dist-tags.js .............................. 26/26
ok test/03-star.js .................................... 10/10
ok test/04-deprecate.js ................................. 7/7
ok test/04b-registry-views-expect.js .................... 1/1
ok test/04b-registry-views.js ......................... 51/51
ok test/04c-lists.js .................................... 3/3
ok test/05-unpublish.js ............................... 24/24
ok test/06-not-implemented.js ........................... 9/9
ok test/common.js ....................................... 2/2
ok test/ping.js ......................................... 9/9
ok test/pkg-update-copy-fields.js ....................... 3/3
ok test/pkg-update-deprecate-msg.js ..................... 3/3
ok test/pkg-update-hoodie.js ............................ 2/2
ok test/pkg-update-metadata.js .......................... 4/4
ok test/pkg-update-star-nopt.js ......................... 4/4
ok test/pkg-update-yui-error.js ......................... 2/2
ok test/vdu-auth.js ..................................... 7/7
ok test/vdu.js ........................................ 16/16
ok test/zz-teardown.js .................................. 2/2
total ............................................... 246/247

not ok
npm ERR! Test failed.  See above for more details.
```

I'm stumped! Anyone got any ideas?
