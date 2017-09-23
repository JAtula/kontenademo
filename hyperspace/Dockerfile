FROM golang:1.9-alpine
RUN apk update && apk add git 
WORKDIR /home/root/hyperspace
ADD . .
RUN touch tmp/server.pid && \
    go get github.com/gorilla/websocket && \
    go get github.com/lucasb-eyer/go-colorful && \
    cd server && \
    go build
EXPOSE 9393
ENTRYPOINT ./server/server -port 9393 echo $$! > tmp/server.pid