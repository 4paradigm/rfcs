- Start Date: 2021-04-23
- Target Major Version: (2.2.0)
- Reference Issues: 
- Implementation PR: 

# Add API Server

## Summary

HTTP interface is one of the most friendly interface and many developer like using http interface to develop their program. HTTP API also help us to show demo simply. So we want to support uing http api to access FEDB.

## Detailed design

### Interface design
HTTP URL: http://ip:port/sql

HTTP Method: POST

#### Put
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

#### Execute procedure 
```
{
    "method": "execute_procedure",
    "db": "db_name",
    "table": "table_name",
    "value": {
        "name": "procedure_name"
    }

}
```
#### Execute SQL

```
{
    "method": "execute_sql",
    "db": "db_name",
    "value": {
        "sql": "select * from t1"
    }

}
```

### Server design

We use [brpc HTTP Service](https://github.com/apache/incubator-brpc/blob/master/docs/en/http_service.md) framework to tackle the http request. So what need to do is forwarding the reques to FEDB server with fedb sdk in brpc http service implementation.
