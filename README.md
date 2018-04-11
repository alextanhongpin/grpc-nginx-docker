# traefik-grpc
gRPC load balancing with Nginx. The README is heavily inspired from [nginx docs](https://www.nginx.com/blog/nginx-1-13-10-grpc/).

## Prerequisite

As gRPC needs HTTP2, we need valid HTTPS certificates on both gRPC Server and Nginx.

## Creating Nginx Certificate

The important thing is the subject must be set to `nginx`, which is the name of the nginx service:

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./cert/nginx.key -out ./cert/nginx.cert  -subj '/CN=nginx'
```


## Nginx Configuration

At last, we configure our Træfik instance to use both self-signed certificates.


```nginx
user nginx;

worker_processes auto;

worker_rlimit_nofile 10240;

# Leave this empty for now
events {}

http {
	log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
					  '$status $body_bytes_sent "$http_referer" '
					  '"$http_user_agent"';

	map $http_upgrade $connection_upgrade {
		default upgrade;
		''        close;
	}

	upstream grpcservers {
		# The docker endpoint of your grpc servers, you can have multiple here
		server server1:50051;
		server server2:50052;
	}

	server {
		listen 1443 ssl http2;

		# Create a certificate that points to the hostname, e.g. nginx for docker
		# $ openssl req -nodes -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -subj '/CN=nginx'
		ssl_certificate     /run/secrets/nginx.cert;
		ssl_certificate_key /run/secrets/nginx.key;

		location /echo.EchoService {
			# Replace localhost:50051 with the address and port of your gRPC server
			# The 'grpc://' prefix is optional; unencrypted gRPC is the default
			grpc_pass grpcs://grpcservers;
		}
	}
}
```


## gRPC Server Example

```go
// ...

// Read cert and key file
BackendCert, _ := ioutil.ReadFile("./nginx.cert")
BackendKey, _ := ioutil.ReadFile("./nginx.key")

// Generate Certificate struct
cert, err := tls.X509KeyPair(BackendCert, BackendKey)
if err != nil {
  log.Fatalf("failed to parse certificate: %v", err)
}

// Create credentials
creds := credentials.NewServerTLSFromCert(&cert)

// Use Credentials in gRPC server options
serverOption := grpc.Creds(creds)
var s *grpc.Server = grpc.NewServer(serverOption)
defer s.Stop()

pb.RegisterGreeterServer(s, &server{})
err := s.Serve(lis)

// ...
```

## gRPC Client Example

```go
// ...

// Read cert file
FrontendCert, _ := ioutil.ReadFile("./frontend.cert")

// Create CertPool
roots := x509.NewCertPool()
roots.AppendCertsFromPEM(FrontendCert)

// Create credentials
credsClient := credentials.NewClientTLSFromCert(roots, "")

// Dial with specific Transport (with credentials)
conn, err := grpc.Dial("nginx:1443", grpc.WithTransportCredentials(credsClient))
if err != nil {
    log.Fatalf("did not connect: %v", err)
}

defer conn.Close()
client := pb.NewGreeterClient(conn)

name := "World"
r, err := client.SayHello(context.Background(), &pb.HelloRequest{Name: name})

// ...
```

## Build

If you have not build the docker images, you can execute this command to build it. Yes, you can set it in docker-compose to build the image, but I prefer to separate it.

```bash
$ make build-server

$ make build-client
```

## Test

Run this several time to trigger the `client`. The `client` is not running a server, so the docker image is not persistent.

The IP changes:

```bash
# Start the application
$ docker-compose up -d

# Make the client call the server
$ docker-compose up -d client
```

## Client Response

To view the response from the `client`:

```bash
$ docker logs $(docker ps -a --filter name=client -q)
```

Output:

```
2018/04/10 07:22:46 got res: &echo.EchoResponse{Text:"hello world from f553b776babf"}
2018/04/10 07:28:52 got res: &echo.EchoResponse{Text:"hello world from c07dec651f40"}
2018/04/10 07:28:56 got res: &echo.EchoResponse{Text:"hello world from f553b776babf"}
```

## Nginx Response

```bash
$ docker logs $(docker ps -a --filter name=nginx -q)
```

Output:

```bash
172.18.0.2 - - [10/Apr/2018:07:22:46 +0000] "POST /echo.EchoService/Echo HTTP/2.0" 200 36 "-" "grpc-go/1.12.0-dev"
172.18.0.2 - - [10/Apr/2018:07:28:52 +0000] "POST /echo.EchoService/Echo HTTP/2.0" 200 36 "-" "grpc-go/1.12.0-dev"
172.18.0.2 - - [10/Apr/2018:07:28:56 +0000] "POST /echo.EchoService/Echo HTTP/2.0" 200 36 "-" "grpc-go/1.12.0-dev"
```