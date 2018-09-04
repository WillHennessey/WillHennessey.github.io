---
layout: post
title: Extract and parse digital certificates using Ruby
description: Extracting and parsing digital certificates from PE files using Ruby
keywords: extracting, parsing, digital, certificates, ruby, pe, files
comments: true
date: 2017-03-12 21:35:00 +0000
---
<div class="message">
    Recently I came across a challenging piece of work. 
    I was required to write a pure Ruby solution to extract and parse digital certificates from PE files. 
</div>

## Reading in .der and .pem files

Before I cover how to extract digital certificates from a PE file let's take a look at a standard PKCS7 certificate in .der format. 
First I read in my_certificate.der and create an OpenSSL::PKCS7 object from the raw certificate string.
I then instantiate the DigitalCertificateParser class with the PKCS7 object as a parameter.

<script src="https://gist.github.com/WillHennessey/15cddfc5fd5e932779bc97e4c7ea02de.js"></script>

## Extracting X509 certificates from a PKCS7 certificate

Digital signatures are usually a collection of X509 certificates within a PKCS7 certificate, the X509 certificates are the ones we're interested in.
The parser pulls these OpenSSL::X509 certificates out from the PKCS7 certificate on line 5. 
I can now call `@digital_certificate_parser.chains` to build an array containing one or more ordered arrays of digital certificates, 
from the root certificate to the end user certificate, with intermediate certificates in between.

The logic I've implemented here is based on 'name match' validation which is explained very well in [this article](https://sites.google.com/site/ddmwsst/digital-certificates). 
In a nutshell certificates in a chain are linked by their subject and issuer values. 

The issuer value of a end user certificate should be the subject of the next certificate in the chain, this continues up the chain to the root certificate.
The root certificate is the certificate that does not have it's issuer referenced by any other certificate as it is a loop back to itself. 

<script src="https://gist.github.com/WillHennessey/5744f84ff3f7647dbb2e9c3465a99661.js"></script>

## Decorating the certificate's data

You'll notice the parser is creating an object called `@all_certificates` when it's initialized.
This is basically mapping all the X509 certificates to become DigitalCertificate objects.
The DigitalCertificate object is one I've written, it uses the Decorator design pattern by inheriting from the [SimpleDelegator class](https://ruby-doc.org/stdlib-2.1.0/libdoc/delegate/rdoc/SimpleDelegator.html).

This allows me to add functions like `time_stamping_certificate?` and `to_db` to the X509 certificates.
I can also overwrite functions that the OpenSSL::X509 object would usually respond to like `subject` with the subject value decorated, in this case as a string.

The Decorator pattern also allows me to call functions like `to_der`, `to_pem` and `verify`, so essentially what we have is an X509 certificate object with some extra functionality I've added. 
 
<script src="https://gist.github.com/WillHennessey/9a96b28903494f248358406c571c934b.js"></script>

Digital signatures will usually include a timestamp countersignature, when working with certificates I wasn't concerned with these so I chose to identify them and ignore them. 
If you are interested in them you can remove the check in this class and the check in the DigitalCertificateParser class.

## Extracting certificates from a PE file

To extract one or more certificates from a PE file I'm going to use the gem [pedump](https://github.com/zed-0xff/pedump). 
If you're using Bundler then just add this gem to your Gemfile, otherwise you can install it with `gem install pedump`. 
The gem has a command-line implementation, but this didn't suit my requirements so I used pedump's code instead.

>*I have observed one caveat when using the pedemp gem, it is unable to extract SHA256 digital certificates if all certificates in the chain are signed with the sha256 algorithm.*

<script src="https://gist.github.com/WillHennessey/7c996c70514b8f7e4a856dce8d12aef3.js"></script>

This class is just an example class, to show you how you can pull digital certificates from PE files.
You might want to extract all the certificates instead of just the end user certificate, all you have to do just write a function in this class that calls `parser.chains`.
The certificate objects that are returned can be decorated any way you like by modifying the DigitalCertificate class to suit your needs. 

## Putting it all together

So far the code covers parsing certificates from .der and .pem files, 
extracting certificates from PE files and decorating certificates to display information in a way you want.

The last snippet I'll share is some examples of how to use this code.
Feel free to modify the code to suit your needs. 

<script src="https://gist.github.com/WillHennessey/28dac43cc2313a8e70d9a9365831748d.js"></script>