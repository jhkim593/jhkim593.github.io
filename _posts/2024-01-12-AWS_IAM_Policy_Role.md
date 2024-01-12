---
layout: post
title: "AWS - IAM Policy 와 Role"
author: "jhkim593"
tags: AWS

---
본 장에서는 IAM을 구성하는 요소인 Policy(정책)와 Role(역할)와 좀 더 세부적인 요소인Trust Relationships (신뢰관계), Permissions boundary( 권한경계)에 대해서 다뤄보겠습니다.

# IAM

AWS 리소스에 대한 액세스를 안전하게 제어할 수 있는 웹서비스이며 AWS 리소스에 대한 액세스 허용 및 거부 권한 설정을 관리한다.

IAM을 구성하는 요소는 크게 다음과 같습니다.
- Policy (정책)
- Role (역할)
- User (사용자)
- UserGroup (사용자 그룹)

<br>

# Policy (정책)

AWS 서비스에 대한 허용 규칙을 정의합니다. JSON 형식으로 작성되며 Role( 역할 )과 IAM 사용자에게 부여 가능합니다.

정책을 쉽게 풀어서 적는다면

**“(어떤 Principal이) (어떤 Condition을 만족할 때,) 어떤 Resource에 대해서 어떤 Action을 할 수 있게 허용/불허(Effect)한다.” 로 정리 할 수 있다.**

가장 많이 활용되는 정책은 **자격증명-기반 정책(Id-Based Policy) , 리소스-기반 정책(Resource-Based Policy)** 두가지가 있습니다.

<br>

## **자격증명-기반 정책(Id-Based Policy)**

aws 계정 , iam 사용자 , 역할 등이 보안 주체 ( Principal )이 되기 때문에 Principal 속성이 없습니다.

json 형식 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
       ],
      "Resource": "arn:aws:s3:::test.net"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
      ],
      "Resource": "arn:aws:s3:::test.net/*"
    }
  ]
}
```

- Effect: 허용인지 불허인지 나타냅니다.
- Action: 특정 서비스에 어떤 작업인지 정의합니다.
- Resource: 대상이 되는 resource를 나타냅니다.
- Condition ( Optional ) : 해당 보안 주체가 만족해야하는 조건을 설정합니다.

각 서비스 마다 고유 **Resource**가 있는데. s3를 예로들면 리소스는 버킷과 객체가 될 수 있습니다.  

그렇기 때문에 동일한 서비스라고 하더라도 여러 Statement로 나누어서 정책이 설정될 수 있습니다.

위 예시에서 `s3:ListBucket` 은 버킷에 대한 액션이기 때문에 R**esource**로 버킷을 가르키고있습니다.

 `s3:GetObject` 은 객체에 대한 액션이기 때문에 **Resource**로 특정 버킷의 모든 객체를 가르키고 있습니다.

`s3:ListAllMyBuckets` 은 리소스를 한정하지않는 액션이기 때문에 **Resource**가 *로 설정되어있습니다.

<br>

## **리소스-기반 정책(Resource-Based Policy)**

자격 증명 기반정책은 IAM 에서 관리하지만 리소스 기반 정책은 일반적으로 서비스에서 따로 관리하게됩니다.

json 형식 예시

```json
{
    "Version": "2008-10-17",
    "Id": "PolicyForCloudFrontPrivateContent",
    "Statement": [
        {
            "Sid": "1",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E36E6CTWWW86E5"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::test.net/*"
        }
    ]
}
```

자격 증명 기반 정책과 다르게 Princiapl 속성이 들어가야합니다.

**CloudFront Origin Access Identity E36E6CTWWW86E5 ( Principal )** 에 test.net에 업로드된 모든 객체를 읽을 수 있는 권한을 부여하는 것입니다.

<br>

# Role ( 역할 )

AWS 서비스 액세스를 위한 하나 이상의 Policy (정책)기반으로 구성됩니다.

 IAM 사용자나 IAM 그룹에는 연결되지 않으며 다른 AWS 계정 , AWS 서비스에 설정해 권한을 지정할 수 있습니다.

역할은 정책을 갖는다는점에서 IAM 사용자와 비슷하지만 누가 이 Role을 맡을 수 있는지 Principal을 지정하기 위한 **Trust Relationships (신뢰 관계)** 가 추가되어야 합니다.

<br>
## Trust Relationships (신뢰 관계)

역할에 설정되며 역할을 맡을 수 있는 주체와 조건을 정의합니다. 이를 IAM 역할에 대한 리소스 기반 정책이라고도 합니다.

JSON 형식으로 작성됩니다.

신뢰 관계 JSON 예시

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "eks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

위 예시는 특정 역할에 정의된 신뢰관계 예시입니다.

Principa (주체)가 AssumeRole 하는것을 allow (허용 )되는 것이며 즉 AWS EKS가 이 role을 사용할 수 있게 된것입니다.

<br>

# 권한 경계

**IAM 사용자 또는 역할**이 가질 수 있는 최대 권한을 설정할 수 있어 권한 허용 범위를 제한할 수 있습니다.

권한 경계를 설정하게되면 정책과의 교집합이되는 부분에 작업만 수행할 수 있습니다.

<img src="https://github.com/jhkim593/ChawChawBack/assets/53510936/cf968178-de0d-4445-959f-1cdef8660927"  width="250" height="250"/>

예를들어

IAM user가 가지고 있는 정책중 EC2 FullAccess 정책이 있더라도 권한 경계에 Ec2ReadOnlyAccess만 있다면 교집합인 ReadOnlyAccess만 가능합니다.
