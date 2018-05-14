# Adding a custom SSL certificate to Amazon CloudFront
In this tutorial we will be taking an existing SSL certificate and adding to cloudfront so that we can serve out content via HTTPS and a custom domain. My own experience of this inspired me to write everything I found here because there are a few caveat's that are not covered everywhere.

## Setting up
The first things you will need is an SSL certificate. They can be purchased from various places on the internet and I am not going to support any providers in this tutorial.

Secondly you will need to create a new DNS record in AWS. This will be used when setting up the CNAME for the CloudFront instance.

Next we will need to create an AWS user with all the permissions required to upload the SSL certificate. I created a user and gave it all IAM permissions, here is how: <%link_to_other_tutorial%>

After that we need to install the AWS CLI. This requires Python and the Python package manager PIP. As I am using OSX I can use Homebrew to install both.

### Installing AWS CLI

```
brew update && brew install python
```

This will install both Python and PIP, so then we can install the AWS CLI.

```
pip install awscli --upgrade --user
```

We can test the validation of the installation by running the command:

```
aws --version
```

Which should give us an output of something like this...

```
aws-cli/1.15.10 Python/3.6.5 Darwin/17.5.0 botocore/1.10.10
```

Now we have the CLI installed we can add the users credentials to our profile and begin to use it. Depending on which OS you are using this might be in a different file (bash_profile, bash_rc).

```
cd ~
vi .bash_profile
```

Export the credentials for the user, making the replacements as required.

```
# AWS exports
export AWS_ACCESS_KEY_ID="<%KEY%>"
export AWS_SECRET_ACCESS_KEY="<%SECRET%>"
export AWS_DEFAULT_REGION="<%REGION%>"
```

We should now have access to all the IAM features via the CLI. This will help us install the SSL certificate for the CloudFront instance, so let's get to it.

### Uploading a SSL certificate to IAM

To upload a certificate, we are going to use the IAM upload function via the CLI:

```
aws iam upload-server-certificate --server-certificate-name <%SAVE_AS_ON_AWS%> --certificate-body file://<%PATH_TO_PUBLIC_CERT%> --private-key file://<%PATH_TO_PRIVATE_CERT%> --certificate-chain file://<%PATH_TO_CHAIN_CERT%> --path /cloudfront/
```

If you are anything like me then this confused the hell out of me, which file is which and what the hell is a _chain file_? Fear not, I am here to explain the difference, albeit very simply.

`public key/cert` - this is the compiled certificate, which is public and downloaded by the browsers so that they know that the data that is being transfered actually came from the server issuing the certificate.

`private key/cert` - this is the key that has been used to sign the certificate and is private to the server issuing the public certificate. This means when the browser validates the connection using the public certificate/key it knows that the connection is valid and the data has not been tampered with.

`chain key/cert` - this is the intermediate key provided from the certificate issuer. The content of these files are your root certificate, followed by any intermediate certificates, followed by your generated SSL certificate. Usually these are issued when downloading your certificates and are usually the __.crt__ file. A guide to creating them yourself can be found [here](https://support.comodo.com/index.php?/Knowledgebase/Article/View/1145/1/how-do-i-make-my-own-bundle-file-from-crt-files).

#### Uploading Gotcha's

The first thing to note when uploading these files via the CLI is that all the file paths need to be fixed with `file://`. This is very important and is well documented, but important nonetheless, because it makes the CLI look for a local version of the file that you are trying to upload. If you forgot this prefix chances are that you would get a message that looks like this one:

```
A client error (MalformedCertificate) occurred when calling the UploadServerCertificate operation: Unable to parse certificate. Please ensure the certificate is in PEM format.
```

If you are still recieving this error when you upload the files with the prefix, then it is quite possible that the files you have are not RSA encoded and this is another stipulation for uploading the files.

To change a DER private key to RSA format, run this command:

```
openssl rsa -inform DER -in PrivateKey.der -outform PEM -out PrivateKey.pem
```

To change a chain certificate from DER to RSA, run this command:

```
openssl x509 -inform DER -in Certificate.der -outform PEM -out Certificate.pem
```

For other options, there are some options in the AWS documentation [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_server-certs.html#server-certificate-troubleshooting).

I tried all of the above and started to get the following error:

```
An error occurred (MalformedCertificate) when calling the UploadServerCertificate operation: Unable to validate certificate chain. The certificate chain must start with the immediate signing certificate, followed by any intermediaries in order. The index within the chain of the invalid certificate is: -1
```

Recreating the chain file was providing a variation in this message, where the integer for the index was changing, but I could not fix it. My faith was running out until I tried this:

```
aws iam upload-server-certificate --server-certificate-name <%SAVE_AS_ON_AWS%> --certificate-body file://<%PATH_TO_PUBLIC_CERT%> --private-key file://<%PATH_TO_PRIVATE_CERT%> --path /cloudfront/
```

Yes! Despite all the documentation, the chain file is not actually required and once I removed it there was no problem uploading the SSL certificate! Hours I spent trying to format the files and re-download them to compile in different ways.

#### Creating a CloudFront instance

Now all we need to complete the tutorial is a CloudFront instance, so log into the AWS console and click CloudFront.

Choose the __Create Distribution__ option and you will be given two options, __Web__ and __RTMP__. We are going to set up a __Web__ distribution so select that from the options, this will then take you to a settings page to create the distribution.

Here the first option, the __Origin Domain Name__, is a pointer to the S3 bucket that you wish to use and will usually provide options to all the buckets that are currently available to the signed in user.

Select your bucket and then move to the __Origin Path__. This is where the magic starts, if you are just exposing the whole bucket then you can skip this part, otherwise, this is where we can tie the CloudFront instance to a specific path.

Let's pretend we have a bucket called _thebucketwithahole.com.s3.amazonaws.com_ and inside this bucket there is a path `some-directory/another-directory/a-file-to-expose.txt`. Here is where we can expose that file or a directory in the path. To do so fill out the _Origin Path_ with the path that you want to expose starting with a /.

```
/some-directory/another-directory/a-file-to-expose.txt
```

The next parameter we are going to address in this tutorial is the __Alternate Domain Names__ parameter. This is where we can add the DNS we already set up so that it will serve out the CloudFront instance from that URI. Complete this field with the domain you set up in AWS earlier.

Now we have a custom domain, we can select our SSL certificate that we uploaded using the __Custom SSL Certificate__ radio button. If this option is not yet highlighted for some reason, wait for a few minutes to let the SSL certificate propegate and then start this section from the beginning, it's just because it needs time to be recognised by the console.

## Congratulations
Save the CloudFront instance and give it a little while to build and propegate and you should now have a fully functioning CloudFront instance which is accessible via HTTPS on your domain.