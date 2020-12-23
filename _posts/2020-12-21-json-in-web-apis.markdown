---
layout: post
title:  "JSON in web apis - a practical guide"
date:   2020-12-23 12:51:15 +0100
categories:
---
[Json][json] has become the default message format for any new web api. It's simple enough to just start coding and it does not require a schema which is handy while exploring ideas. But a time comes when the web api becomes accessible to users and they need to know what to send. The backend service should also be able to distinguish between valid and invalid messages. Here are some ideas how to approach this.

# JSON schema
There is a [json schema][json schema] enhancement available for defining json syntax rules similarly like in xml. A sample schema looks like this:
{% highlight json %}
{
  "$id": "https://example.com/email.schema.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "EmailInfo",
  "type": "object",
  "required": [ "email" ],
  "properties": {
    "email": {
      "type": "string",
      "format": "email",
      "description": "Users email."
    }
  },
  "additionalProperties": false
}
{% endhighlight %}

A sample json inline with this schema would be:
{% highlight json %}
{
  "email":"lbd01@protonmail.com"
}
{% endhighlight %}

With the **required** property you can select which properties *have to* be inside the json.

The newer (draft-06+) schema drafts allow to specify the expected **format** of the property value - in the provided example it's an email.

Lastly the **additionalProperties** property (true by default) decides if properties not defined in the schema are allowed. Because in the schema example I set this property to false the following json would be invalid:

{% highlight json %}
{
  "email":"lbd01@protonmail.com",
  "phone":"+44 123 123 123"
}
{% endhighlight %}


The nice thing about json schema is that it's implementation agnostic and can be easily shared with external parties. There are many tools that allow you to generate code out of json schema definition. In **Java** for generating pojo classes out of schema you can use [Json Schema 2 Pojo][json schema 2 pojo]. To validate jsons against a schema you can use [Json Schema Validator][json schema validator]. See [here][json schema implementation] for a list of tools for other languages.

# OpenAPI / Swagger
[OpenAPI][open api] (known previously as Swagger) is a standard for creating api specifications. It also allows to define accepted json message formats. As a matter of fact it's actually based on the previously mentioned json schema. Two OpenAPI versions are in use and both are based on a *pre draft-06* json schema specification.
- [OpenAPI v2][open api v2] - based on [draft-zyp-04][draft-zyp-04]
- [OpenAPI v3][open api v3] - based on [draft-wright-00][draft-wright-00]


The **format** property was introduced in draft-06 but OpenAPI defined this property on it's own. It is possible to provide value format hints (even completely custom ones).


A sample api specification together with json message format definition in OpenAPI v3 looks like this:
{% highlight yaml %}
openapi: "3.0.0"
info:
  title: "Email api"
  version: "1.0.0"
paths:
  /send-email:
    post:
      summary: Send a test email
      responses:
        200:
          description: status
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/EmailStatus"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/EmailRequest"
components:
  schemas:
    EmailRequest:
      type: object
      required:
        - email
      properties:
        email:
          type: string
          format: email
    EmailStatus:
      type: object
      properties:
        success:
          type: boolean
{% endhighlight %}


In this api the user can submit an email address and the backend will try to send a test message to it. If it's successful it will return true and false otherwise.

In OpenAPI the default behavior is to allow properties unspecified in schema which is equivalent to `additionalProperties:true`. In V2 it's not possible to set `additionalProperties:false`. See [here][open api v2 additionalProperties] for more details.


The [swagger editor][swagger editor] allows to generate code out of OpenAPI specifications both for client and server side. Be aware that because of the aforementioned json schema difference the **format** validations might be missing from the code.

# Making changes / Versioning
First let's distinguish 2 types of changes in a message definition:
- *incremental* - where new elements are added but all the existing ones are left intact,
- *breaking* - where existing elements get refactored / deleted.

Following the previous examples an *incremental* change would look like this (new field added):
{% highlight json %}
{
  "email":"lbd01@protonmail.com",
  "topic":"My test email"
}
{% endhighlight %}
A *breaking* change could be something like this (email replaced with 2 new fields):
{% highlight json %}
{
  "username":"lbd01",
  "domain":"protonmail.com"
}
{% endhighlight %}


In [semantic versioning][semantic versioning] the *incremental change* would be a *MINOR* release and the *breaking change* would be a *MAJOR* release (MAJOR.MINOR.PATCH).


To ensure that the api user doesn't have to recompile their code whenever you publish a *MINOR* release make sure to define your messages with the `additionalProperties:true` property. This way the user will be aware that new json properties might be added in the future and upgrade at it's own pace.


Let's go back to the previous OpenAPI example and make an incremental change in the Email message types.
{% highlight yaml %}
openapi: "3.0.0"
info:
  title: "Test email api"
  version: "1.1.0"
paths:
  /send-email:
    post:
      summary: Send a test email
      responses:
        200:
          description: email sending status
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/EmailStatus"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/EmailRequest"
components:
  schemas:
    EmailRequest:
      type: object
      required:
        - email
      properties:
        email:
          type: string
          format: email
        topic:
          type: string
    EmailStatus:
      type: object
      properties:
        success:
          type: boolean
        completed:
          type: string
          format: date-time
{% endhighlight %}

Few things changed:
- the version is now 1.1.0 (*MINOR* release),
- a new optional property `topic` was added in the request,
- a new optional property `completed` was added in the response.

V1.1.0 Request:
{% highlight json %}
{
  "email":"lbd01@protonmail.com",
  "topic":"My test email"
}
{% endhighlight %}


V1.1.0 Response:
{% highlight json %}
{
  "success":true,
  "completed":"2020-12-21T14:07:54.958Z"
}
{% endhighlight %}

As mentioned earlier in OpenAPI additional properties are allowed by default. This means that the v1.1.0 response is compliant with the api v1.0.0. On the other side: because `topic` is optional the v1.0.0 request is compliant with api v1.1.0.

If the `topic` property becomes required in v1.1.0 then the requests coming from v1.0.0 user will no longer be compliant with v1.1.0. An obligatory property will be missing.


If the `completed` property becomes required in v1.1.0 it will have no consequences to the v1.0.0 user (all additional properties are accepted in v1.0.0).

# Handling multiple api versions

It's rarely the case when you can force your api users to upgrade to a newer api version. Even if you can do it most likely there will be a transition period needed. This means a period in which multiple api versions are supported simultaneously. A good example of this is [google drive api][google drive api]. Currently v2 and v3 are supported whereas v1 is no longer available.

There are at least 2 possible ways of handling multiple api versions:

- **Option 1**: Deploying new api version as a separate service
- **Option 2**: Supporting multiple api versions in one service

**Option 1** is a 'brute force' approach. It will probably mean that 2 code bases will modify the same database which needs to be handled with caution. Besides this (and a bigger infrastructure bill) it's pretty straightforward. There are 2 api versions available: `current` and `new` one. The users need to migrate to the `new` version and when they do the `old` is shutdown.

Personally I've seen option 1 used only in big companies with controlled environments. *(Well quite often I've also seen an alternative strategy where the api version is changed with all it's users at once without any transition period but with a couple hours downtime on all involved software. Mostly on weekends. Mostly on internal (not customer facing) software. If after update one component failed then all the others were rollbacked as well.)*

If you're dealing with b2b partners that use your api or some mobile apps then the prospects of the user updating to a newer api version are slim.

For the b2b partner doing additional IT development might be costly, inconvenient and simply not his priority. And with the mobile app all the updates might be blocked by the user.

The problem might get multiplied by the number of versions you need to support. *User 1* could have started with version 1, *User 2* with version 2. Version 3 is now available but both refuse to update. Going with Option 1 in this situation might be too costly and difficult to maintain. This is where **Option 2** comes in.

In **Option 2** a single service is capable of handling multiple api versions. With this approach a problem arises: *How to recognize which api version the user is calling?* Here are some potential solutions:
- by url
- by payload content
- by content-type

# Handling multiple api versions by url

In this approach everytime a new api version is introduced it becomes available under a separate url. The url usually contains the api version e.g. <https://www.googleapis.com/drive/v3>.

In OpenAPI v3 you can do this by adding a `servers` section into the api specification. (In OpenAPI v2 there is a `basePath` property).

For our v1.0.0 api example we could add the following:

{% highlight yaml %}
servers:
  - url: https://api.lbd01.net/email-test/v1
{% endhighlight %}

For v1.1.0 it could be e.g.:

{% highlight yaml %}
servers:
  - url: https://api.lbd01.net/email-test/v1.1
{% endhighlight %}

In the backend you can easily distinguish which version was called by looking at the http invoke path.

If the new api versions are released often (*MAJOR* and *MINOR*) it might be inconvenient for the api users to constantly switch the urls. Taking this into account you might want to reserve this technique only for *MAJOR* releases.

# Handling multiple api versions by payload content

If the changes made in the api's jsons are purely incremental (*MINOR*) you could try to identify the api version by looking at the received payload. If you design your api's payloads around distinct features it might not be even necessary to know what exact version you're handling. You just either support or not a certain feature.

For the email sender api example this could look like this (Java):
{% highlight java %}
EmailRequest request = JsonUtil.parseJson(httpRequestPayload, EmailRequest.class);
emailSender.send(
  request.getEmail(),
  request.getTopic()!=null?request.getTopic():"Default topic"
  );

{% endhighlight %}

The `topic` feature was introduced in v1.1.0 and in the calls made by v1.0.0 users it will remain empty. Nevertheless it's easy enough to write code that can handle both versions. Obviously this approach heavily depends on the api specifics so YMMV. The potential gain here is less code repetition and less errors.


# Handling multiple api versions by content-type

If you make a POST or PUT the [Content Type][content type] header specifies the type of data that is being sent. Usually it's just `application/json` (sometimes empty). The [RFC6838][RFC6838] allows to provide custom content-type values with the vendor (vnd.), personal (prs.) or unregistered (x.) prefixes. There is some mentioning of a registration process but I don't think everyone follows it.

Custom content type can be used to convey information about api version. E.g. [Openbanking UK Branches API][openbanking branches api] uses `application/prs.openbanking.opendata.v2.3+json` media type with the api version baked into it. If we apply this approach to our email api example we will receive something like this:

{% highlight yaml %}
openapi: "3.0.0"
info:
  title: "Test email api"
  version: "1.1.0"
paths:
  /send-email:
    post:
      summary: Send a test email
      responses:
        200:
          description: email sending status
          content:
            application/prs.net.lbd01.email.v1.1+json:
              schema:
                $ref: "#/components/schemas/EmailStatus"
      requestBody:
        required: true
        content:
          application/prs.net.lbd01.email.v1.1+json:
            schema:
              $ref: "#/components/schemas/EmailRequest"
components:
  schemas:
    EmailRequest:
      type: object
      required:
        - email
      properties:
        email:
          type: string
          format: email
        topic:
          type: string
    EmailStatus:
      type: object
      properties:
        success:
          type: boolean
        completed:
          type: string
          format: date-time
{% endhighlight %}

Note that the content type changed from `application/json` to `application/prs.net.lbd01.email.v1.1+json`. Obviously in v1.0.0 api the version would be v1.

By analyzing the http request headers it's possible to determine which api version was invoked and how to deserialize the received payload. The caveat here is that it's not a standard approach and might require few lines of code more on the api user side as well as in the backend (configure support for custom media types). Some of these shortcomings can be probably worked around by using a different header than Content-Type (e.g. X-API-VERSION) but this is even more uncommon.


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

[json]: https://www.json.org/json-en.html
[json schema]: https://json-schema.org/
[json schema implementation]: https://json-schema.org/implementations.html
[json schema validator]: https://github.com/everit-org/json-schema
[json schema 2 pojo]: https://github.com/joelittlejohn/jsonschema2pojo
[open api]: https://swagger.io/resources/open-api/
[open api v2]: https://swagger.io/specification/v2/
[open api v3]: https://swagger.io/specification/
[open api v2 additionalProperties]: https://support.reprezen.com/support/solutions/articles/6000162892-support-for-additionalproperties-in-swagger-2-0-schemas
[draft-zyp-04]: https://tools.ietf.org/html/draft-zyp-json-schema-04
[draft-wright-00]: https://tools.ietf.org/html/draft-wright-json-schema-00
[swagger editor]: https://github.com/swagger-api/swagger-editor
[semantic versioning]: https://semver.org
[openbanking branches api]: https://openbanking.atlassian.net/wiki/spaces/DZ/pages/1103233386/Branch+API+Specification+v2.3.0
[google drive api]: https://developers.google.com/drive
[content type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type
[RFC6838]: https://tools.ietf.org/html/rfc6838#section-3
