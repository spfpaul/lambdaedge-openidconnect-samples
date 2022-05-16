# Amazon CloudFront and Lambda@Edge OIDC Function

## Note

This repo is cloned from https://github.com/aws-samples/lambdaedge-openidconnect-samples, with following update in template.yaml:

1. Update LogBucketName.DomainName to **LoggingBucket**.DomainName in CFDistribution

```bash
    CFDistribution:
    ...
            Logging:
              Bucket: !GetAtt LoggingBucket.DomainName
```

2. Append * at the end of Resource in policy in LambdaEdgeFunctionRole

```bash
    LambdaEdgeFunctionRole:
    ...
                    Resource: !Sub "arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:${SecretKeyName}*"
```

## Steps

1. Create a Secret in Secret Manager with key "config" and dummy value

2. Update src/js/okta-key.txt with the name of Secret created above

3. Export environment variables
```bash
export DEPLOY_BUCKET="YOUR_SAM_S3_BUCKET"
export STATIC_WEBSITE_BUCKET="YOUR_STATIC_ASSET_BUCKET"
export YOUR_LOG_BUCKET_NAME="YOUR_LOG_BUCKET_NAME"
export YOUR_SECRETS_MANAGER_KEY_NAME="YOUR_SECRETS_MANAGER_KEY_NAME"
```

4. Build and deploy the package

```bash
sam build -b ./build -s . -t template.yaml -u

sam package \
 --template-file build/template.yaml \
 --s3-bucket ${DEPLOY_BUCKET} \
 --output-template-file build/packaged.yaml

sam deploy \
 --template-file build/packaged.yaml \
 --stack-name oidc-auth \
 --capabilities CAPABILITY_NAMED_IAM \
 --parameter-overrides BucketName=${STATIC_WEBSITE_BUCKET} LogBucketName=${YOUR_LOG_BUCKET_NAME} SecretKeyName=${YOUR_SECRETS_MANAGER_KEY_NAME}
```

5. Create OIDC App integration in Okta with Sign-in redirect URL configured as "https://CLOUDFRONT_DIST_URL/_callback"


6. Generate Public and Private keys
```bash
openssl genrsa -out private.pem 2048
openssl rsa -in key.pem -outform PEM -pubout -out public.pem
awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' private.pem;echo
awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' public.pem;echo
```

7. Update the config below with Okta config, CloudFront distribution and Public/Private key generated above

```json
{
  "AUTH_REQUEST": {
  	"client_id": "${CLIENT_ID_FROM_OKTA}",
  	"response_type": "code",
  	"scope": "openid email",
  	"redirect_uri": "https://${CLOUDFRONT_DIST_URL}/_callback"
  },
  "TOKEN_REQUEST": {
  	"client_id": "${CLIENT_ID_FROM_OKTA}",
  	"redirect_uri": "https://${CLOUDFRONT_DIST_URL}/_callback",
  	"grant_type": "authorization_code",
  	"client_secret": "${CLIENT_SECRET_FROM_OKTA}"
  },
  "DISTRIBUTION": "amazon-oai",
  "AUTHN": "OKTA",
  "PRIVATE_KEY": "${PRIVATE_KEY_GOES_HERE}",
  "PUBLIC_KEY": "${PUBLIC_KEY_GOES_HERE}",
  "DISCOVERY_DOCUMENT": "https://${OKTA_DOMAIN_NAME}/.well-known/openid-configuration",
  "SESSION_DURATION": 30,
  "BASE_URL": "https://${OKTA_DOMAIN_NAME}/",
  "CALLBACK_PATH": "/_callback",
  "AUTHZ": "OKTA"
}
  ```

8. Encode the config above to Base64 format with online tool, and replace the dummy Key value of the Secret created above