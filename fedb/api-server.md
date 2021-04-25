- Start Date: 2021-04-23
- Target Major Version: (2.2.0)
- Reference Issues: 
- Implementation PR: 

# Add API Server

# Summary

HTTP interface is one of the most friendly interface and many developer like using http interface to develop their program. HTTP API also help us to show demo simply. So we want to support uing http api to access FEDB.

# Detailed design


## Interface design
HTTP URL: http://ip:port/sql

HTTP Method: POST

### Put
request
```
{
    "method": "put",
    "db": "db_name",
    "table": "table_name",
    "value": {
        "field1": "value1",
        "field2": 111,
        "field3": 1.4   
    }

}
```
response
```
{
    "code": 0,
    "msg": "ok"
}
```

### Execute procedure 
reqeust
```
{
    "method": "execute_procedure",
    "db": "db_name",
    "table": "table_name",
    "value": {
        "name": "procedure_name",
        "input": {
            "id":123,
            "field1": "value1"
        }
    }

}
```
response
```
{
    "code": 0,
    "msg": "ok",
    "data": [{"field1" : "value1"}, {"field1": "value2"}]
}
```
### Execute SQL
request

```
{
    "method": "execute_sql",
    "db": "db_name",
    "data": {
        "sql": "select * from t1"
        "input": {
            "id":123,
            "field1": "value1"
        }
    }

}
```
response
```
{
    "code": 0,
    "msg": "ok",
    "data": [{"field1" : "value1"}, {"field1": "value2"}] 
}
```

## Server design

We use [brpc HTTP Service](https://github.com/apache/incubator-brpc/blob/master/docs/en/http_service.md) framework to tackle the http request. So what need to do is forwarding the reques to FEDB server with fedb sdk in brpc http service implementation.

1. Design proto file.  
    ```
    option cc_generic_services = true;

    message HttpRequest { };
    message HttpResponse { };

    service APIService {
        rpc ForwardSQL(HttpRequest) returns (HttpResponse);
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
        virtual void ForwardSQL(google::protobuf::RpcController* controller,
                                const HttpRequest* request,
                                HttpResponse* response,
                                google::protobuf::Closure* done) {
            // parse request attachment
            ...
        }
    };
    ```
3. Dispatch and forward request
    
    ```
    if (method == "put") {
        // sdk put
    } else if (method == "execute_procedure") {
        // sdk execute procedure
    } else if (method == "execute_sql") {
        // sdk execute sql
    } eles {
        LOG(ERROR) << "unsupport method";
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