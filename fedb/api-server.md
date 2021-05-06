- Start Date: 2021-04-23
- Target Major Version: (2.2.0)
- Reference Issues: 
- Implementation PR: 

# Add API Server

# Summary

HTTP interface is one of the most friendly interface and many developers like using http interface to develop their program. REST API also help us to show demo simply. So we want to support uing http api to access FEDB.

# Detailed Design

## Interface Design

### Schema Mapping
|FEDB type|Json type|
|---|---|
|string|String|
|smallint|Number|
|int|Number|
|bigint|Number|
|float|Number|
|double|Number|
|bool|Boolean|
|null|null|
|date|String|
|timestamp|String|

JSON does not have a built-in type for date/timestamp values. We store the date/timestamp value as a string in [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format, e.g 2021-05-06, 2021-05-06T00:02:32Z.

### HTTP Status 
|code|message|
|--|--|
|200|ok|
|404|page not found|

### Put
reqeust url: http://ip:port/db/{db_name}/table/{table_name}  
http method: PUT
request body: 
```
{
    "value": [{
        "field1": "value1", // string
        "field2": 111,  // int
        "field3": 1.4,  // float/double
        "field4": "2021-04-27"  // date
        "field5": "2021-04-27T08:03:15+00:00"  //timestamp
        "field6": true // bool
    }]

}
```
response
```
{
    "code": 0,
    "msg": "ok"
}
```

|code|message|
|--|--|
|0|ok|
|1|get insert information failed|

### Execute Procedure 
reqeust url: http://ip:port/db/{db_name}/procedure/{procedure_name}  
http method: POST  
request body: 
```
{
    "common_cols":["value1", "value2"],
    "input": [["value1", "value2"],["value1", "value2"]],
    "need_schema": true
}
```
response
```
{
    "code": 0,
    "msg": "ok",
    "data": {
        "schema": [{"field1":"bool"}, {"field2":"int32"}],
        "data": [["value1", "value2"], [...]]
    }
}
```
|code|message|
|--|--|
|0|ok|
|-1|execute procedure failed|

### Get Procedure
request url: http://ip:port/db/{db_name}/procedure/{procedure_name}   
http method: Get
response
```
{
    "code": 0,
    "msg": "ok",
    "data":{
        "name": "procedure_name",
        "procedure": "xxxxx",
        "common_col": ["field1", "field2"],
        "input_schema": [{"name":"field1", "type":"bool"}, {"name":"filed1", "type":"int32"}],
        "output_schema": [{"field1": "bool"}, {"field2": "int32"}],
        "tables": ["table1", "table2"]
    }
}
```
|code|message|
|--|--|
|0|ok|
|-1|procedure not found|

## Server design

We use [brpc HTTP Service](https://github.com/apache/incubator-brpc/blob/master/docs/en/http_service.md) framework to tackle the http request. So what need to do is forwarding the reques to FEDB server with fedb sdk in brpc http service implementation.

1. Design proto file.  
    ```
    option cc_generic_services = true;

    message HttpRequest { };
    message HttpResponse { };

    service APIService {
        rpc ProcessTable(HttpRequest) returns (HttpResponse);
        rpc ProcessProcedure(HttpRequest) returns (HttpResponse);
    }
    ```
2. Implement the service  
    ```
    class APIServiceImpl : public APIService {
      public:
        bool Init() {
            // init the sdk
            ...
        }
        virtual void ProcessTable(google::protobuf::RpcController* controller,
                                const HttpRequest* request,
                                HttpResponse* response,
                                google::protobuf::Closure* done) {
            swich (cntl->http_request().method()) {

                case HTTP_METHOD_PUT):
                    // parse request attachment and put data
                    ...
                    break;
                default:
                    // log error
                    break;
            }
        }
        virtual void ProcessProcedure(google::protobuf::RpcController* controller,
                                const HttpRequest* request,
                                HttpResponse* response,
                                google::protobuf::Closure* done) {
            switch (cntl->http_request().method()) {
                case brpc::HTTP_METHOD_GET:
                    // get procedure info
                    break;
                case brpc::HTTP_METHOD_POST:
                    // execute procedure
                    break;
                default:
                    // log error
                    break;
            }
        }
    };
    ```
3. Add mapping
    
    ```
    if (server.AddService(&api_svc,
                      brpc::SERVER_DOESNT_OWN_SERVICE,
                      "/db/*/table     => ProcessTable,"
                      "/db/*/procedure => ProcessProcedure") != 0) {
        LOG(ERROR) << "Fail to add api service";
        return -1;
    }
    ```

### JSON lib
We use [RapidJSON](https://github.com/Tencent/rapidjson) as json parser lib for the following reasons:
- RapidJSON is fast.
- RapidJSON is self-contained and header-only. It does not depend on external libraries such as BOOST. 

Benchmark details can be read [here](https://rawgit.com/miloyip/nativejson-benchmark/master/sample/performance_Corei7-4980HQ@2.80GHz_mac64_clang7.0.html#1.%20Parse)

# Alternative
Set parameters in url, ex [postgrest](https://github.com/PostgREST/postgrest).
```
GET /people?age=gte.18&student=is.true HTTP/1.1
```
For complete document, refer to [here](https://postgrest.org/en/stable/api.html#horizontal-filtering-rows).

This method make API Interfaces more complicated.

# Adoption stratege
If we implement this proposal, there is no impact on existing interfaces.