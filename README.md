# AWS Maven Wagon
This project is a [Maven Wagon][wagon] for [Amazon S3][s3].  In order to to publish artifacts to an S3 bucket, the user (as identified by their access key) must be listed as an owner on the bucket.

## Usage
To publish Maven artifacts to S3 a build extension must be defined in a project's `pom.xml`.  The latest version of the wagon can be found on the [`aws-maven`][aws-maven] page in Maven Central.

```xml
<project>
  ...
  <build>
    ...
    <extensions>
      ...
      <extension>
        <groupId>org.springframework.build</groupId>
        <artifactId>aws-maven</artifactId>
        <version>5.0.0.RELEASE</version>
      </extension>
      ...
    </extensions>
    ...
  </build>
  ...
</project>
```

Once the build extension is configured distribution management repositories can be defined in the `pom.xml` with an `s3://` scheme.

```xml
<project>
  ...
  <distributionManagement>
    <repository>
      <id>aws-release</id>
      <name>AWS Release Repository</name>
      <url>s3://<BUCKET>/release</url>
    </repository>
    <snapshotRepository>
      <id>aws-snapshot</id>
      <name>AWS Snapshot Repository</name>
      <url>s3://<BUCKET>/snapshot</url>
    </snapshotRepository>
  </distributionManagement>
  ...
</project>
```

Finally the `~/.m2/settings.xml` must be updated to include access and secret keys for the account. The access key should be used to populate the `username` element, and the secret access key should be used to populate the `password` element.

```xml
<settings>
  ...
  <servers>
    ...
    <server>
      <id>aws-release</id>
      <username>0123456789ABCDEFGHIJ</username>
      <password>0123456789abcdefghijklmnopqrstuvwxyzABCD</password>
    </server>
    <server>
      <id>aws-snapshot</id>
      <username>0123456789ABCDEFGHIJ</username>
      <password>0123456789abcdefghijklmnopqrstuvwxyzABCD</password>
    </server>
    ...
  </servers>
  ...
</settings>
```

Alternatively, the access and secret keys for the account can be provided using

* `AWS_ACCESS_KEY_ID` (or `AWS_ACCESS_KEY`) and `AWS_SECRET_KEY` (or `AWS_SECRET_ACCESS_KEY`) [environment variables][env-var]
* `aws.accessKeyId` and `aws.secretKey` [system properties][sys-prop]
* The Amazon EC2 [Instance Metadata Service][instance-metadata]

## Making Artifacts Public
This wagon doesn't set an explict ACL for each artfact that is uploaded.  Instead you should create an AWS Bucket Policy to set permissions on objects.  A bucket policy can be set in the [AWS Console][console] and can be generated using the [AWS Policy Generator][policy-generator].

In order to make the contents of a bucket public you need to add statements with the following details to your policy:

| Effect  | Principal | Action       | Amazon Resource Name (ARN)
| ------- | --------- | ------------ | --------------------------
| `Allow` | `*`       | `ListBucket` | `arn:aws:s3:::<BUCKET>`
| `Allow` | `*`       | `GetObject`  | `arn:aws:s3:::<BUCKET>/*`

If your policy is setup properly it should look something like:

```json
{
  "Id": "Policy1397027253868",
  "Statement": [
    {
      "Sid": "Stmt1397027243665",
      "Action": [
        "s3:ListBucket"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::<BUCKET>",
      "Principal": {
        "AWS": [
          "*"
        ]
      }
    },
    {
      "Sid": "Stmt1397027177153",
      "Action": [
        "s3:GetObject"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::<BUCKET>/*",
      "Principal": {
        "AWS": [
          "*"
        ]
      }
    }
  ]
}
```

If you prefer to use the [command line][cli], you can use the following script to make the contents of a bucket public:

```bash
BUCKET=<BUCKET>
TIMESTAMP=$(date +%Y%m%d%H%M)
POLICY=$(cat<<EOF
{
  "Id": "public-read-policy-$TIMESTAMP",
  "Statement": [
    {
      "Sid": "list-bucket-$TIMESTAMP",
      "Action": [
        "s3:ListBucket"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::$BUCKET",
      "Principal": {
        "AWS": [
          "*"
        ]
      }
    },
    {
      "Sid": "get-object-$TIMESTAMP",
      "Action": [
        "s3:GetObject"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::$BUCKET/*",
      "Principal": {
        "AWS": [
          "*"
        ]
      }
    }
  ]
}
EOF
)

aws s3api put-bucket-policy --bucket $BUCKET --policy "$POLICY"
```

[aws-maven]: http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.springframework.build%22%20AND%20a%3A%22aws-maven%22
[cli]: http://aws.amazon.com/documentation/cli/
[console]: https://console.aws.amazon.com/s3
[env-var]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/EnvironmentVariableCredentialsProvider.html
[instance-metadata]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/InstanceProfileCredentialsProvider.html
[policy-generator]: http://awspolicygen.s3.amazonaws.com/policygen.html
[s3]: http://aws.amazon.com/s3/
[sys-prop]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/SystemPropertiesCredentialsProvider.html
[wagon]: http://maven.apache.org/wagon/

## Usage in deploy:deploy-file
If you'd like to use aws-maven via `mvn deploy:deploy-file` to deploy non-maven jars to your s3 maven repo, you might want to install the aws-maven wagon provider and all of its dependencies into your maven installation's lib directory, so that the aws-maven wagon provider is available outside of the context of a project and its pom.xml. Here's one way to do that:

1. Uncomment the section in this project's pom.xml containing the shaded jar build.

2. Build this project

		mvn clean package
 or if you find that an integration test fails
 
 		mvn -DskipTests=true clean package
 	
3. Copy the shaded jar into your $MAVEN_HOME/lib

 First, find your maven home:

		$ mvn -v
		Apache Maven 3.2.3 (33f8c3e1027c3ddde99d3cdebad2656a31e8fdf4; 2014-08-11T16:58:10-04:00)
		Maven home: /usr/local/Cellar/maven/3.2.3/libexec
		Java version: 1.7.0_40, vendor: Oracle Corporation
		Java home: /Library/Java/JavaVirtualMachines/jdk1.7.0_40.jdk/Contents/Home/jre
		Default locale: en_US, platform encoding: UTF-8
		OS name: "mac os x", version: "10.9.5", arch: "x86_64", family: "mac"
	
 Now copy the shaded jar into your maven home:

		cp target/aws-maven-wagon.jar /usr/local/Cellar/maven/3.2.3/libexec/lib
	
4. Deploy your non-maven jar using a deploy:deploy-file command similar to this:
	
		mvn deploy:deploy-file \
		    -Dfile=example.jar \
		    -DgroupId=com.example \
		    -DartifactId=example \
		    -Dversion=1.0.0 \
		    -Dpackaging=jar \
		    -DrepositoryId=aws-release \
		    -Durl=s3://<BUCKET>/release
