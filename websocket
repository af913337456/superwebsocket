package main

import (
	"errors"
	"fmt"
	"net/http"
	"strings"
	"sync"
	"time"

	"github.com/pborman/uuid"
)
import "github.com/gorilla/websocket"

const keepLive = `{"type":"ping"}`

type MyWsClient struct {
	WsUrl           string
	ClientId        string
	Password        string
	readMsgErrMark  bool
	connectedOnce   sync.Once
	reConnectSignal chan string
	Heartbeat       time.Duration
	reConnectLock   sync.Mutex
	WsConn          *websocket.Conn
	HandleMsg       func(msg []byte)
}

func NewMyWsClient(url, clientId string) *MyWsClient {
	return &MyWsClient{
		WsUrl:           url + clientId,
		WsConn:          nil,
		ClientId:        clientId,
		connectedOnce:   sync.Once{},
		reConnectSignal: make(chan string),
		Heartbeat:       10 * time.Second,
		reConnectLock:   sync.Mutex{},
		readMsgErrMark:  false,
	}
}

func (u *MyWsClient) SetHandleMsg(handleMsg func(msg []byte)) {
	u.HandleMsg = handleMsg
}

func (u *MyWsClient) Connect(password string, connected func()) error {
	header := http.Header{}
	header.Set("password", password)
	conn, _, err := websocket.DefaultDialer.Dial(u.WsUrl, header)
	if err != nil {
		return fmt.Errorf("ConnectWs url: %s, err: %v", u.WsUrl, err)
	}
	if err = conn.WriteMessage(websocket.PingMessage, []byte(keepLive)); err != nil {
		return fmt.Errorf("ws PingMessage err: %v", err)
	}
	if connected != nil {
		connected()
	}
	u.Password = password
	u.WsConn = conn
	u.connectedOnce.Do(func() {
		go u.wsHeartbeat()
		go u.wsHandleMsg(func() {
			reconnectUUID := uuid.NewRandom().String()
			u.reConnectSignal <- reconnectUUID
		})
		go func() {
			for {
				select {
				case rUUid := <-u.reConnectSignal:
					u.WsConn = nil
					for {
						if u.WsConn != nil {
							u.log(fmt.Sprintf("重连成功! %s", rUUid))
							break
						}
						u.log(fmt.Sprintf("ws已经断开，[%s] 准备重连...", rUUid))
						time.Sleep(time.Second * 2)
						u.ReConnect("from server err, uuid:" + rUUid)
					}
				}
			}
		}()
	})
	return nil
}

func (u *MyWsClient) log(msg string) {
	connect := false
	if u.WsConn != nil {
		connect = true
	}
	fmt.Println(fmt.Sprintf("%s, ReqTime: %s --> %s, ws!=nil: %v", u.logPrefix(), time.Now().Format("2006-01-02 15:04:05"), msg, connect))
}

func (u *MyWsClient) logPrefix() string {
	return fmt.Sprintf("ClientId:%s", u.ClientId)
}

func (u *MyWsClient) wsHeartbeat() {
	ping := func() error {
		if u.WsConn == nil {
			return nil
		}
		if _err := u.WsConn.WriteMessage(websocket.TextMessage, []byte(keepLive)); _err != nil {
			u.log(fmt.Sprintf("ConnectWs, heartbeat err: %v", _err))
			if strings.Contains(_err.Error(), "broken pipe") || strings.Contains(_err.Error(), "close sent") {
				go func() {
					reconnectUUID := uuid.NewRandom().String()
					u.log("wsHeartbeat ---> reConnectSignal_" + reconnectUUID)
					u.reConnectSignal <- reconnectUUID // 重连
					u.log("wsHeartbeat ---> reConnectSignal BeenRead_" + reconnectUUID)
				}()
				return errors.New("close broken pipe")
			}
		} else {
			// u.log(time.Now().String() + "---> send ping success")
		}
		return nil
	}
	for {
		if err := ping(); err != nil {
			u.log(fmt.Sprintf("ping err: %v", err))
			time.Sleep(u.Heartbeat)
			continue
		}
		time.Sleep(u.Heartbeat)
	}
}

func (u *MyWsClient) disconnect(a, b int64) {
	u.reConnectLock.Lock()
	defer u.reConnectLock.Unlock()
	_ = u.WsConn.Close()
	u.WsConn = nil
	u.log(fmt.Sprintf("err 主动断开连接, Current: %d, LastTime: %d", a, b))
}

func (u *MyWsClient) wsHandleMsg(disconnectHandle func()) {
	defer func() {
		if r := recover(); r != nil {
			// repeated read on failed websocket connection
			u.log(fmt.Sprintf("wsHandleMsg panic: %v", r))
		}
	}()
	for {
		if u.WsConn == nil {
			time.Sleep(time.Second)
			continue
		}
		if _, message, err := u.WsConn.ReadMessage(); err != nil {
			u.log(fmt.Sprintf("ws ReadMessage err: %v", err))
			if isDisconnectErr(err) && disconnectHandle != nil {
				// 重连
				disconnectHandle()
				time.Sleep(time.Second)
			} else if isSelfCloseErr(err) {
				//disconnectHandle()
				u.readMsgErrMark = true
				time.Sleep(time.Millisecond * 500)
			}
		} else {
			if u.readMsgErrMark {
				u.log(fmt.Sprintf("ws ReadMessage err 恢复正常"))
			}
			u.readMsgErrMark = false
			//u.log(fmt.Sprintf("ws 接收消息: %v", string(message)))
			if u.HandleMsg != nil {
				u.HandleMsg(message)
			}
		}
	}
}

func (u *MyWsClient) SendMsg(msg []byte) error {
	if u.WsConn == nil {
		return errors.New("u.WsConn == nil")
	}
	return u.WsConn.WriteMessage(websocket.TextMessage, msg)
}

func (u *MyWsClient) ReConnect(logTip string) {
	u.reConnectLock.Lock()
	defer u.reConnectLock.Unlock()
	if u.WsConn != nil {
		u.log("u.WsConn != nil 不重连, " + logTip)
		return
	}
	if cErr := u.Connect(u.Password, func() {

	}); cErr != nil {
		u.log(fmt.Sprintf("%s %s", cErr.Error(), logTip))
	}
}

func isDisconnectErr(err error) bool {
	return strings.Contains(err.Error(), "close 1006")
}

// use of closed network connection
func isSelfCloseErr(err error) bool {
	return strings.Contains(err.Error(), "use of closed network connection")
}

func shouldReconnect(err error) bool {
	if strings.Contains(err.Error(), "connection refused") ||
		strings.Contains(err.Error(), "unexpected EOF") ||
		strings.Contains(err.Error(), "bad handshake") {
		return true
	}
	return false
}
