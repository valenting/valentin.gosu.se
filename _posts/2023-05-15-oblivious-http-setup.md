---
layout: post
title: "Setup and Oblivious HTTP gateway locally"
---

# What is Oblivious HTTP

Oblivious HTTP is an [IETF draft](https://datatracker.ietf.org/doc/draft-ietf-ohai-ohttp/) for a system that allows you to fetch a resource from a web server without the server being able to identify who fetched that resource.

![Image](/OHTTP_diagram.png)

This post contains instructions on how to set up a relay and gateway for local testing and fetch a resource from the web through the Gateway using Firefox.

# Set up gateway

```bash
# clone the repo
git clone https://github.com/cloudflare/privacy-gateway-server-go.git
cd privacy-gateway-server-go

# create a self-signed certificate
sudo apt install mkcert
mkcert -key-file key.pem -cert-file cert.pem 127.0.0.1 localhost

# install the required packages for go (Ubuntu Linux)
sudo apt install protobuf-compiler protoc-gen-go golang-go
make all

# run the gateway
CERT=cert.pem KEY=key.pem PORT=4567 ./gateway
```

# Set up a relay

```bash
# clone relay repo
git clone https://github.com/valenting/nodejs-ohttp-relay.git
cd nodejs-ohttp-relay/

# install nodejs if you don't already have it
sudo apt install nodejs

# run the relay
node index.js
```

# Open browser and accept certificate

To do this, open Firefox and navigate to https://localhost:4567/ohttp-configs
Accept the certificate warning.

# OHTTP code

Run the following code to perform an OHTTP request.

```js
(async () => {
  var ohttpService = Cc[
    "@mozilla.org/network/oblivious-http-service;1"
  ].getService(Ci.nsIObliviousHttpService);

  var ohttp = Cc["@mozilla.org/network/oblivious-http;1"].getService(
    Ci.nsIObliviousHttp
  );

  var relayURI = NetUtil.newURI(
    `http://localhost:8080/`
  );
  var requestURI = NetUtil.newURI("https://mozilla.com/");

  // https://searchfox.org/mozilla-central/rev/d8c5478bd730644dab4336fdb83271ac1ca1d6b5/netwerk/protocol/http/ObliviousHttpService.cpp#49

  var ohttpServer = ohttp.server();
  var config = ohttpServer.encodedConfig;
  var config = await fetch(
    "https://localhost:4567/ohttp-configs"
  );
  console.log("fetched");
  config = await config.blob();
  config = await config.arrayBuffer();
  config = new Uint8Array(config);
  // config = [...config];

  console.log(
    "Requesting",
    config,
    btoa(bytesToString(config)),
    requestURI.spec.toString()
  );
function read_stream(stream, count) {
  /* assume stream has non-ASCII data */
  var wrapper = Cc["@mozilla.org/binaryinputstream;1"].createInstance(
    Ci.nsIBinaryInputStream
  );
  wrapper.setInputStream(stream);
  /* JS methods can be called with a maximum of 65535 arguments, and input
     streams don't have to return all the data they make .available() when
     asked to .read() that number of bytes. */
  var data = [];
  while (count > 0) {
    var bytes = wrapper.readByteArray(Math.min(65535, count));
    data.push(String.fromCharCode.apply(null, bytes));
    count -= bytes.length;
    if (!bytes.length) {
      do_throw("Nothing read from input stream!");
    }
  }
  return data.join("");
}

  function bytesToString(bytes) {
    if (bytes.length > 65535) {
      throw new Error("input too long for bytesToString");
    }
    return String.fromCharCode.apply(null, bytes);
  }

  var obliviousHttpChannel = ohttpService
    .newChannel(relayURI, requestURI, config)
    .QueryInterface(Ci.nsIHttpChannel);

  var headers = {
    "X-Some-Header": "header value",
    "X-Some-Other-Header": "25",
  };

  for (var headerName of Object.keys(headers)) {
    obliviousHttpChannel.setRequestHeader(
      headerName,
      headers[headerName],
      false
    );
  }

  let listener = {
      _buffer: "",
      QueryInterface: ChromeUtils.generateQI([
        "nsIStreamListener",
        "nsIRequestObserver",
      ]),
      onStartRequest(request) {},
      onDataAvailable(request, stream, offset, count) {
          this._buffer += read_stream(stream, count);
      },
      onStopRequest(request, status) {
          console.log("onStop", this._buffer);
      }
  };
  obliviousHttpChannel.asyncOpen(listener);
})();
```