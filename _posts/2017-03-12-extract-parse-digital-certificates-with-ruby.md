---
layout: post
title: Extract and parse digital certificates using Ruby
description: Extracting and parsing digital certificates from PE files using Ruby
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

<div class = "block-code">
{% highlight ruby %}
raw_pkcs7_cert = File.read(Rails.root + 'my_certificate.der')
certificate_object = OpenSSL::PKCS7.new(raw_pkcs7_cert)
@digital_certificate_parser = DigitalCertificateParser.new(certificate_object) 
{% endhighlight %}
</div>

## Extracting X509 certificates from a PKCS7 certificate

Digital signatures are usually a collection of X509 certificates within a PKCS7 certificate, the X509 certificates are the ones we're interested in.
The parser pulls these OpenSSL::X509 certificates out from the PKCS7 certificate on line 5. 
I can now call `@digital_certificate_parser.chains` to build an array containing one or more ordered arrays of digital certificates, 
from the root certificate to the end user certificate, with intermediate certificates in between.

The logic I've implemented here is based on 'name match' validation which is explained very well in [this article](https://sites.google.com/site/ddmwsst/digital-certificates). 
In a nutshell certificates in a chain are linked by their subject and issuer values. 

The issuer value of a end user certificate should be the subject of the next certificate in the chain, this continues up the chain to the root certificate.
The root certificate is the certificate that does not have it's issuer referenced by any other certificate as it is a loop back to itself. 

<div class = "block-code">
{% highlight ruby %}

class DigitalCertificateParser
  attr_reader :all_certificates

  def initialize(digital_signature)
    if !digital_signature.instance_of?(OpenSSL::PKCS7)
      raise InvalidSignatureError, 'Signature is not a PKCS7'
    end
    @all_certificates = digital_signature.certificates.map do |cert| 
      DigitalCertificate.new(cert) 
    end
  end

  def chains
    @chains ||= build_chains
  end

  def end_user_certificate
    first_chain = chains[0]
    first_chain.last if first_chain
  end

  def root_certificate
    first_chain = chains[0]
    first_chain.first if first_chain
  end

  private

  def build_chains
    chains = []
    root_certs = find_root_certs
    root_certs.each_with_index do |cert, index|
      chains[index] = [cert]
      while (next_certificate = find_next_in_chain(chains[index].last))
        chains[index] << next_certificate
      end
    end
    chains
  end

  def find_next_in_chain(parent_cert)
    all_certificates.find do |cert| 
      cert.issuer == parent_cert.subject && !cert.time_stamping_certificate?
    end
  end

  def find_root_certs
    subjects = all_certificates.map(&:subject)
    all_certificates.select do |certificate| 
      certificate.root_certificate?(subjects.delete(certificate.subject))
    end
  end
end
{% endhighlight %}
</div>

## Decorating the certificate's data

You'll notice the parser is creating an object called `@all_certificates` when it's initialized.
This is basically mapping all the X509 certificates to become DigitalCertificate objects.
The DigitalCertificate object is one I've written, it uses the Decorator design pattern by inheriting from the [SimpleDelegator class](https://ruby-doc.org/stdlib-2.1.0/libdoc/delegate/rdoc/SimpleDelegator.html).

This allows me to add functions like `time_stamping_certificate?` and `to_db` to the X509 certificates.
I can also overwrite functions that the OpenSSL::X509 object would usually respond to like `subject` with the subject value decorated, in this case as a string.

The Decorator pattern also allows me to call functions like `to_der`, `to_pem` and `verify`, so essentially what we have is an X509 certificate object with some extra functionality I've added. 
 
<div class="block-code">
{% highlight ruby %}
require 'delegate'
require 'date'

class DigitalCertificate < SimpleDelegator
  def subject
    __getobj__.subject.to_s
  end

  def issuer
    __getobj__.issuer.to_s
  end

  def serial
    __getobj__.serial.to_s(16).scan(/.{1,2}/).join(' ')
  end

  def extensions
    __getobj__.extensions.map(&:value)
  end

  def valid_from
    __getobj__.not_before.to_s
  end

  def valid_to
    __getobj__.not_after.to_s
  end

  def thumbprint
    OpenSSL::Digest::SHA1.new(__getobj__.to_der).to_s.upcase
  end

  def root_certificate?(subjects)
    !subjects.include?(issuer) && !time_stamping_certificate?
  end

  def time_stamping_certificate?
    extension_match = extensions.any? { |ext| ext =~ /time\s*stamp/i }
    extension_match || subject =~ /time\s*stamp/i
  end

  def to_db
    { issuer: issuer, subject: subject, serial_no: serial, 
      thumbprint: thumbprint, algorithm: signature_algorithm, 
      valid_from: DateTime.parse(valid_from), 
      valid_to: DateTime.parse(valid_to) 
    }
  end
end
{% endhighlight %}
</div>

Digital signatures will usually include a timestamp countersignature, when working with certificates I wasn't concerned with these so I chose to identify them and ignore them. 
If you are interested in them you can remove the check in this class and the check in the DigitalCertificateParser class.

## Extracting certificates from a PE file

To extract one or more certificates from a PE file I'm going to use the gem [pedump](https://github.com/zed-0xff/pedump). 
If you're using Bundler then just add this gem to your Gemfile, otherwise you can install it with `gem install pedump`. 
The gem has a command-line implementation, but this didn't suit my requirements so I used pedump's code instead.

>*I have observed one caveat when using the pedemp gem, it is unable to extract SHA256 digital certificates if all certificates in the chain are signed with the sha256 algorithm.*

<div class="block-code">
{% highlight ruby %}
require 'pedump'

class InvalidFilePathError < StandardError; end

class DigitalCertificateExtractor

  def new(path)
    @file_path = Pathname(path)
    raise InvalidFilePathError unless @file_path.exist?
    @signature = nil
  end
  
  def extract_end_user_certificate
    begin
      File.open(@file_path, 'rb') do |f| 
        @signature = PEdump.new(f, :log_level => Logger::UNKNOWN ).signature 
      end
      return nil unless @signature && @signature.any?
      parser = DigitalCertificateParser.new(@signature.first.data)
      parser.end_user_certificate
    rescue InvalidSignatureError => e
      puts "File with invalid signature: #{@file_path}\n\n" + e.backtrace.join("\n")
      nil
    end
  end
end
{% endhighlight %}
</div>

This class is just an example class, to show you how you can pull digital certificates from PE files.
You might want to extract all the certificates instead of just the end user certificate, all you have to do just write a function in this class that calls `parser.chains`.
The certificate objects that are returned can be decorated any way you like by modifying the DigitalCertificate class to suit your needs. 

## Putting it all together

So far the code covers parsing certificates from .der and .pem files, 
extracting certificates from PE files and decorating certificates to display information in a way you want.

The last snippet I'll share is some examples of how to use this code.
Feel free to modify the code to suit your needs. 

<div class="block-code">
{% highlight ruby %}
# Extracting a certificate chain from a PKCS7 file
raw_pkcs7_cert = File.read('/path/to/my_pkcs7.der')
certificate_object = OpenSSL::PKCS7.new(raw_pkcs7_cert)
parser = DigitalCertificateParser.new(certificate_object)
certificate_chain = parser.build_chains.first
certificate_chain.each |certificate|
  puts "Subject: #{certificate.subject}\nIssuer: #{certificate.issuer}\n\n"
end

# Extracting an end user certificate from a PE file
extractor = DigitalCertificateExtractor.new('path/to/your/file.exe')
certificate = extractor.extract_end_user_certificate

# Store a certificate in a Database using Rails
MyModel.create!(certificate.to_db) unless certificate.nil?

# Write to a file using OpenSSL::X509 functionality
File.open('path/to/my_cert.cer', 'wb') { |f| f.write certificate.to_der }

File.open('path/to/my_cert.pem', 'wb') { |f| f.write certificate.to_pem }
{% endhighlight %}
</div>