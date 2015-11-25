---
layout: post
title:  "A Let's Encrypt Client for Go"
date:   2015-11-13
categories: go tls lets encrypt letsencrypt
---

<small>Just want to jump into the GitHub project? [Click here](https://github.com/ericchiang/letsencrypt).</small>

If you haven't heard, [Let's Encrypt](https://letsencrypt.org/) is trying to secure the internet with automated and free TLS certificates. 

In preperation for the Public Beta in December, I've written a simple client in Go which you can go get on [GitHub](https://github.com/ericchiang/letsencrypt). This post uses that client to take you through the workflow of signing up, completing challenges, and requesting certificates. Let's Encrypt!

For these examples, I've spun up a [Let's Encrypt server](https://github.com/letsencrypt/boulder) in dev mode on my local machine, made some edits to the server's configuration, and added an entry to my `/etc/hosts`. See details [here](https://github.com/ericchiang/letsencrypt#running-the-tests) if you want to follow along. 

## Step 1: Registration

To start the client registers a key pair with the server, then uses the private key to sign all future messages and confirm its identity.

First let's generate a key pair for our account.

```nohighlight
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f letsencrypt -N ''
```

Then we'll load the private key and register it with the server.

```go
package main

import (
	// hint, hint: use goimports to fill in the imports

	"github.com/ericchiang/letsencrypt"
)

func main() {
	// Create a client.
	cli, err := letsencrypt.NewClient("http://localhost:4000/directory")
	if err != nil {
		log.Fatal(err)
	}

	// Load the private key and register with the ACME server.
	key, err := loadKey("letsencrypt")
	if err != nil {
		log.Fatal(err)
	}
	if _, err := cli.NewRegistration(key); err != nil {
		log.Fatal(err)
	}
	fmt.Println("Registration successful!")
}

// loadKey reads and parses a private RSA key.
func loadKey(file string) (*rsa.PrivateKey, error) {
	data, err := ioutil.ReadFile(file)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, errors.New("pem decode: no key found")
	}
	return x509.ParsePKCS1PrivateKey(block.Bytes)
}
```

Nothing too crazy yet.

```nohighlight
$ go run step1.go
Registration successful!
```

## Step 2: Requesting challenges

Once we've registered a key pair the next step is to begin a challenge for a specific domain. In this case we'll be asking for the DNS name `example.org`.

The server responds to an authorization request with a list of challenges supported for that domain and a list of combinations of challenges to perform. We'll print the response and take a look at the results.

```go
auth, _, err := cli.NewAuthorization(key, "dns", "example.org")
if err != nil {
	log.Fatal(err)
}

fmt.Println("Challenges:")
for _, chal := range auth.Challenges {
	fmt.Println(chal.Type, chal.URI)
}
fmt.Println("Combinations:")
for _, comb := range auth.Combs {
	fmt.Println(comb)
}
```

```nohighlight
$ go run step2.go 
Challenges:
simpleHttp http://127.0.0.1:4000/acme/challenge/aOOnrY65LHMSYK6HftGeK795dqEII40NYGVESqqKmGA/1
dvsni http://127.0.0.1:4000/acme/challenge/aOOnrY65LHMSYK6HftGeK795dqEII40NYGVESqqKmGA/2
http-01 http://127.0.0.1:4000/acme/challenge/aOOnrY65LHMSYK6HftGeK795dqEII40NYGVESqqKmGA/3
tls-sni-01 http://127.0.0.1:4000/acme/challenge/aOOnrY65LHMSYK6HftGeK795dqEII40NYGVESqqKmGA/4
Combinations:
[0]
[1]
[2]
[3]
```

The server has issued four different challenges for the domain name.

Since `simpleHttp` and `dvsni` are depricated we care about `http-01` and `tls-sni-01`. The first is a challenge to provision an HTTP response at a given path for the domain, the second requires some trickery with TLS Server Name Identification.

The `Combinations` field displays the set of challenges a client can complete to prove ownership over a domain. In this case, completing any of the challenges is enough.

## Step 3: Complete a challenge

The server has now challenged us to provision either an HTTP or TLS resource. It's our turn to do so then inform the server we're ready.

Let's take a look at the HTTP challenge.

```go
// grab the HTTP challenge again
httpChalURL := "http://127.0.0.1:4000/acme/challenge/aOOnrY65LHMSYK6HftGeK795dqEII40NYGVESqqKmGA/3"
chal, err := cli.Challenge(httpChalURL)
if err != nil {
	log.Fatal(err)
}

// print the challenge
out, err := json.MarshalIndent(&chal, "", "  ")
if err != nil {
	log.Fatal(err)
}
fmt.Printf("%s\n", out)

// get the challenge details
path, resource, err := chal.HTTP(key)
if err != nil {
	log.Fatal(err)
}
fmt.Println("Path:    ", path)
fmt.Println("Resource:", resource)
```

```nohighlight
$ go run step3.go 
{
  "type": "http-01",
  "uri": "http://127.0.0.1:4000/acme/challenge/aOOnrY65LHMSYK6HftGeK795dqEII40NYGVESqqKmGA/3",
  "status": "pending",
  "token": "1UMQfJKJWZwOmisxPTso_nR9tEP_42PHsq3EcibGbtE"
}
Path:     /.well-known/acme-challenge/1UMQfJKJWZwOmisxPTso_nR9tEP_42PHsq3EcibGbtE
Resource: 1UMQfJKJWZwOmisxPTso_nR9tEP_42PHsq3EcibGbtE.wF6w7g9k8byiDJcjhwNpLP883uGpxGZL0NdPTEy5PSc
```

As you can see the server has returned a challenge of type `http-01` and a token. [The spec](https://tools.ietf.org/html/draft-ietf-acme-acme-01#section-7.2) documents how to combine that token with the client's private key into a URL path and a HTTP response.

To actually serve the file, we'll need to spin up a HTTP server on the domain we've requested. We then notify the server that the challenge is ready for verification, and poll the challenge until we get a result.

```go
go func() {
	// serve the resource at the given path
	http.HandleFunc(path, func(w http.ResponseWriter, r *http.Request) {
		io.WriteString(w, resource)
	})
	// Start the HTTP server.
	// The test Let's Encrypt server uses port 5002 instead of 80.
	log.Fatal(http.ListenAndServe(":5002", nil))
}()

if err := cli.ChallengeReady(key, chal); err != nil {
	log.Fatal(err)
}
fmt.Println("You've completed the challenge!")
```

```nohighlight
$ go run step3.go 
You've completed the challenge!
```

And that's it. We've provisioned the resource, the server checked that it was there, and the challenge has been verified.

For examples of completing other challenges, see the [README](https://github.com/ericchiang/letsencrypt#challenges).

## Step 4: Getting a certificate

Once we've completed a set of combinations, the server will finally issue us a certificate. This comes in two steps.

First, the client generates a certificate request. This holds information like which domain we want a certificate for, what public key it will use, and what algorithms the server should use to sign it. Second, we send the certificate request to server and (if all goes well) it will respond with a certificate.

### Step 4.1: Creating a certificate request

You can use tools like `openssl` to generate these request. But since we're talking Go, here's how you make one using the standard library.

```go
package main

import (
	// ...
)

func main() {
	certKey, err := rsa.GenerateKey(rand.Reader, 2048)
	if err != nil {
		log.Fatal(err)
	}

	template := &x509.CertificateRequest{
		SignatureAlgorithm: x509.SHA256WithRSA,
		PublicKeyAlgorithm: x509.RSA,
		PublicKey:          &certKey.PublicKey,
		Subject:            pkix.Name{CommonName: "example.org"},
		DNSNames:           []string{"example.org"},
	}
	csrDER, err := x509.CreateCertificateRequest(rand.Reader, template, certKey)
	if err != nil {
		log.Fatal(err)
	}

	pemEncode := func(b []byte, t string) []byte {
		return pem.EncodeToMemory(&pem.Block{Bytes: b, Type: t})
	}

	keyPEM := pemEncode(x509.MarshalPKCS1PrivateKey(certKey), "RSA PRIVATE KEY")
	csrPEM := pemEncode(csrDER, "CERTIFICATE REQUEST")

	if err := ioutil.WriteFile("example.org.csr", csrPEM, 0644); err != nil {
		log.Fatal(err)
	}
	if err := ioutil.WriteFile("example.org.key", keyPEM, 0644); err != nil {
		log.Fatal(err)
	}
}
```

Running this will create `example.org.csr` and `example.org.key`.

### Step 4.2: Requesting a certificate

Finally we sign the certificate request using our account's key (not the one used to generate the request) and send it to the server. Since the account associated with the key has completed the challenges for the domain we've requested, it should issue us a certificate.

```go
package main

import (
	// ...

	"github.com/ericchiang/letsencrypt"
)

func main() {
	cli, err := letsencrypt.NewClient("http://localhost:4000/directory")
	if err != nil {
		log.Fatal(err)
	}

	// load the account's private key and the certificate request
	key, err := loadKey("letsencrypt")
	if err != nil {
		log.Fatal(err)
	}
	csr, err := loadCSR("example.org.csr")
	if err != nil {
		log.Fatal(err)
	}

	// ask for a new certifiate
	cert, err := cli.NewCertificate(key, csr)
	if err != nil {
		log.Fatal(err)
	}
	data := pem.EncodeToMemory(&pem.Block{Bytes: cert.Raw, Type: "CERTIFICATE"})
	if err := ioutil.WriteFile("example.org.crt", data, 0644); err != nil {
		log.Fatal(err)
	}
}

func loadKey(file string) (*rsa.PrivateKey, error) {
	data, err := ioutil.ReadFile(file)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, errors.New("pem decode: no key found")
	}
	return x509.ParsePKCS1PrivateKey(block.Bytes)
}

func loadCSR(file string) (*x509.CertificateRequest, error) {
	data, err := ioutil.ReadFile(file)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(data)
	if block == nil {
		return nil, errors.New("pem decode: no key found")
	}
	return x509.ParseCertificateRequest(block.Bytes)
}
```

```nohighlight
$ go run step4.go 
$ cat example.org.crt
-----BEGIN CERTIFICATE-----
MIIEVzCCAz+gAwIBAgITAP9Jk5pSbvv6cgNUQrFxf/pzITANBgkqhkiG9w0BAQsF
ADAfMR0wGwYDVQQDExRoYXBweSBoYWNrZXIgZmFrZSBDQTAeFw0xNTExMTMyMDAx
MDBaFw0xNjAyMTEyMDAxMDBaMBYxFDASBgNVBAMTC2V4YW1wbGUub3JnMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwGoE3CugCDyiUflgcaRRi7Zmll39
G4zNMFhsV++rlH5OBb0ChFrEESyQfgH9trNlyRrTSB9qm8JMhDfHS9Fx9taAesYB
XxPGrjcPnWFEgOGZ+CuN6R2akR7LMPuZ4vaoC/y1kMg/ixYa6agsrgoi14OGkOhy
FhwklqsywCUtCBHYDaXVn+Fd4ivukw7nHeBxn5KxdQsItB9EuOukTe1i+fkmvFob
ESigpUO+stDPyKEvuT0WHetTD44MqlBJA2fD2Cl232ZIVr/QZOyUxet1SjcWoWdb
X/BvNpXHvUM+WYggwVRpYea142o/Q2RbttovyhA8fRO2/m6bc2IQryWL0wIDAQAB
o4IBkzCCAY8wDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggr
BgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBT1ynbTWvxbpnI9LR7i1p1p
pt5OhzAfBgNVHSMEGDAWgBT7eE8S+WAVgyyfF380GbMuNupBiTBqBggrBgEFBQcB
AQReMFwwJgYIKwYBBQUHMAGGGmh0dHA6Ly8xMjcuMC4wLjE6NDAwMi9vY3NwMDIG
CCsGAQUFBzAChiZodHRwOi8vMTI3LjAuMC4xOjQwMDAvYWNtZS9pc3N1ZXItY2Vy
dDAWBgNVHREEDzANggtleGFtcGxlLm9yZzAnBgNVHR8EIDAeMBygGqAYhhZodHRw
Oi8vZXhhbXBsZS5jb20vY3JsMGMGA1UdIARcMFowCgYGZ4EMAQIBMAAwTAYDKgME
MEUwIgYIKwYBBQUHAgEWFmh0dHA6Ly9leGFtcGxlLmNvbS9jcHMwHwYIKwYBBQUH
AgIwEwwRRG8gV2hhdCBUaG91IFdpbHQwDQYJKoZIhvcNAQELBQADggEBAHUcscRK
WYkWiTmtJY2cvNdpYU6bYKFhS4Mpkj96MnoWQM9o3ScrLdqdmsH6ObRuK2268M8g
8wH7LqkcUC/4Fn0+nO1b0nSJnZUqtr4l9RvtXceJfIJ8J9/l5MEkWuv/U+V4BuI7
Jo7uOCpEQkh+Rm4SSxRXL0ra69UljUKWkKqf6Ln5Pt74jR8VYBwiBpV9p3aoStwr
waSqKijW4zg5U5n66Ypb5CXRw+0LOtV+U64Fd9ifGBVReNRxvx+k23+3/3YO38YW
rqbQUkp9UsCPuWlxTNXHefV0D/p2SmwAa2No7b7WxXV37nGH+5Uh+rUBsIQ18JOy
IeNqHO/uy/bkmCg=
-----END CERTIFICATE-----
```

Holy moley, it worked.

## Conclusion

This Go client isn't a "one click TLS" tool like Let's Encrypt's existing [command line application](https://github.com/letsencrypt/letsencrypt), but hopefully provides building blocks for similar Go programs. It should also give users the tooling to deal with challenges in custom ways, particularly for those with existing websites that they wish to secure.

The [upcoming Public Beta](https://letsencrypt.org/2015/11/12/public-beta-timing.html) on December 3rd will be a huge step for the Let's Encrypt project and hopefully make HTTPS easier and easier for everyone. Even if you never use the Go package, go check them out!
