## mysql connect pool

## introduce
Connection pooling works by keeping the native connection to the server live when the client disposes of a MySqlConnection. Subsequently, if a new MySqlConnection object is opened, it is created from the connection pool, rather than creating a new native connection. This improves performance.
连接池的概念，当本地创建的connection dispose了，实际上连接并不会断，而是等待一段时间，如果有新的connection调用open，那么会从连接池从返回一个，而不是新创建一个连接去连mysql

## ref
[mysql connection pool](https://dev.mysql.com/doc/connector-net/en/connector-net-connections-pooling.html)
