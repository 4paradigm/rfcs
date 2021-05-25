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
|timestamp|Number|

JSON does not have a built-in type for date/timestamp values. We store the date value as a String in [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format and store the timestamp as a Number, e.g '2021-05-06', 1620471840256.

### HTTP Status 
API Server will return 200 in most case and store the error in the code of reponse's json.

### Put
reqeust url: http://ip:port/dbs/{db_name}/tables/{table_name}  
http method: PUT  
request body: 
```
{
    "value": [[
        "value1",
        111,
        1.4,
        "2021-04-27",
        1620471840256,
        true
    ]]

}
```
response:
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
reqeust url: http://ip:port/dbs/{db_name}/procedures/{procedure_name}  
http method: POST  
request body: 
```
{
    "common_cols":["value1", "value2"],
    "input": [["value0", "value3"],["value0", "value3"]],
    "need_schema": true
}
```
response:
```
{
    "code": 0,
    "msg": "ok",
    "data": {
        "schema": [{"name": "field1", "type": "bool"}, {"name": "field2", "type": "int"}, {"name":"field3", "type": "bigint"}],
        "data": [["value1", "value2"], [...]],
        "common_cols_data": ["value3"]
    }
}
```
|code|message|
|--|--|
|0|ok|
|-1|execute procedure failed|

### Get Procedure
request url: http://ip:port/dbs/{db_name}/procedures/{procedure_name}   
http method: Get  
response:
```
{
    "code": 0,
    "msg": "ok",
    "data":{
        "name": "procedure_name",
        "procedure": "xxxxx",
        "input_schema": [{"name":"field1", "type":"bool"}, {"name":"filed1", "type":"int"}],
        "input_common_cols": ["field1", "field2"],
        "output_schema": [{"name":"field1", "type":"bool"}, {"name":"field2", "type":"int32"}, {"name":"field3", "type":"bigint"}],
        "output_common_cols":["field3"]
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
        rpc ProcessSQL(HttpRequest) returns (HttpResponse);
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
        virtual void Process(google::protobuf::RpcController* controller,
                                const HttpRequest* request,
                                HttpResponse* response,
                                google::protobuf::Closure* done) {
            // parse url and dispatch
        }
    };
    ```
3. Add mapping
    
    ```
    if (server.AddService(&api_svc,
                      brpc::SERVER_DOESNT_OWN_SERVICE,
                      "/dbs/*  => Process" != 0) {
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