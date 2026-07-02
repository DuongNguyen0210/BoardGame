# Realtime Contract

## Tổng Quan

Contract này mô tả protocol giữa Unity client và authoritative server. Transport đề xuất là SignalR nếu backend dùng ASP.NET Core. Nếu đổi transport sang raw WebSocket, contract command/event vẫn nên giữ nguyên.

Server là nguồn dữ liệu chính xác duy nhất. Client chỉ gửi command và render response/event/snapshot.

## Khái Niệm

Command:

- Ý định từ client lên server.
- Có `commandId` để chống xử lý trùng.
- Không có nghĩa là state chắc chắn sẽ đổi.

Request/response:

- Kết quả trực tiếp của command cho requester.
- Có thể accepted, rejected hoặc duplicate.

Event:

- Việc đã xảy ra sau khi server accepted command và state đã đổi.
- Broadcast cho room/lobby hoặc gửi riêng player.

State snapshot:

- Ảnh chụp state hiện tại theo quyền nhìn của một player.
- Dùng khi join room, reconnect, request sync hoặc phát hiện thiếu version.

## Envelope Chung

### Client Command Envelope

```json
{
  "protocolVersion": "1.0",
  "commandId": "cmd_01JZ0000000000000000000001",
  "commandType": "PerformGameAction",
  "playerId": "player_123",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 12,
  "sentAt": "2026-07-02T06:00:00Z",
  "payload": {}
}
```

Rules:

- `commandId` unique theo player trong một khoảng thời gian đủ dài.
- `playerId` trong payload không được tin tuyệt đối; server lấy identity từ connection/session.
- `stateVersion` là version client nghĩ đang nhìn thấy.
- `gameSessionId` có thể null với command lobby/room chưa có game.

### Server Response Envelope

```json
{
  "protocolVersion": "1.0",
  "commandId": "cmd_01JZ0000000000000000000001",
  "status": "Accepted",
  "stateVersion": 13,
  "payload": {}
}
```

### Error Response

```json
{
  "protocolVersion": "1.0",
  "commandId": "cmd_01JZ0000000000000000000001",
  "status": "Rejected",
  "error": {
    "errorCode": "NotYourTurn",
    "message": "Player is not allowed to act in the current turn.",
    "details": {
      "currentTurnPlayerId": "player_456"
    }
  },
  "stateVersion": 12
}
```

Không đưa private cards, deck order hoặc server-only debug data vào `details`.

## Versioning

- `protocolVersion`: version contract realtime.
- `stateVersion`: version game/room state visible trong room.
- Server tăng `stateVersion` sau mỗi mutation accepted.
- Event chứa version mới nhất sau mutation.
- Client bỏ qua event có version cũ hơn local.
- Client gửi `RequestStateSync` nếu phát hiện thiếu version hoặc nhận `StateOutOfDate`.

Backward compatibility:

- Thêm field optional được phép trong minor version.
- Đổi nghĩa field hoặc xóa field cần major version.
- Client phải ignore field lạ.

## ID Chuẩn

- `commandId`: idempotency key do client sinh.
- `roomId`: ID nội bộ server cho room.
- `roomCode`: mã ngắn để join.
- `playerId`: ID player.
- `gameSessionId`: ID ván hiện tại.
- `stateVersion`: số nguyên tăng dần trong room/session.

## Client Commands

### CreateRoom

Tạo phòng mới.

```json
{
  "commandType": "CreateRoom",
  "payload": {
    "gameType": "SimpleHighCard",
    "displayName": "My Room",
    "minPlayers": 2,
    "maxPlayers": 2
  }
}
```

Accepted response:

```json
{
  "status": "Accepted",
  "payload": {
    "roomId": "room_456",
    "roomCode": "ABCD12",
    "hostPlayerId": "player_123",
    "roomState": "Waiting"
  }
}
```

### JoinRoom

Tham gia phòng bằng code hoặc id.

```json
{
  "commandType": "JoinRoom",
  "payload": {
    "roomCode": "ABCD12"
  }
}
```

### LeaveRoom

Rời phòng hiện tại.

```json
{
  "commandType": "LeaveRoom",
  "roomId": "room_456",
  "payload": {}
}
```

### SelectSeat

Chọn seat trong room.

```json
{
  "commandType": "SelectSeat",
  "roomId": "room_456",
  "payload": {
    "seatIndex": 1
  }
}
```

### SetReady

Đổi ready state.

```json
{
  "commandType": "SetReady",
  "roomId": "room_456",
  "payload": {
    "isReady": true
  }
}
```

### StartGame

Host bắt đầu game.

```json
{
  "commandType": "StartGame",
  "roomId": "room_456",
  "payload": {}
}
```

### PerformGameAction

Gửi action trong game.

```json
{
  "commandType": "PerformGameAction",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 5,
  "payload": {
    "actionType": "RevealCard",
    "actionPayload": {
      "cardId": "AS"
    }
  }
}
```

Rule:

- Server kiểm tra `actionType` có được game definition hỗ trợ.
- Server kiểm tra quyền/lượt/state trước khi gọi game engine.
- Client không được gửi kết quả action, winner, dealt cards hoặc deck state.

### RequestStateSync

Yêu cầu snapshot hiện tại.

```json
{
  "commandType": "RequestStateSync",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "payload": {
    "lastKnownStateVersion": 4
  }
}
```

### PlayAgain

Yêu cầu chơi lại sau khi game kết thúc.

```json
{
  "commandType": "PlayAgain",
  "roomId": "room_456",
  "payload": {
    "wantsPlayAgain": true
  }
}
```

## Server Events

### RoomCreated

Gửi tới lobby subscribers hoặc requester.

```json
{
  "eventType": "RoomCreated",
  "roomId": "room_456",
  "stateVersion": 1,
  "payload": {
    "roomCode": "ABCD12",
    "gameType": "SimpleHighCard",
    "playerCount": 1,
    "maxPlayers": 2,
    "roomState": "Waiting"
  }
}
```

### PlayerJoined

```json
{
  "eventType": "PlayerJoined",
  "roomId": "room_456",
  "stateVersion": 2,
  "payload": {
    "playerId": "player_234",
    "displayName": "Guest 234"
  }
}
```

### PlayerLeft

```json
{
  "eventType": "PlayerLeft",
  "roomId": "room_456",
  "stateVersion": 3,
  "payload": {
    "playerId": "player_234",
    "newHostPlayerId": "player_123"
  }
}
```

### SeatChanged

```json
{
  "eventType": "SeatChanged",
  "roomId": "room_456",
  "stateVersion": 4,
  "payload": {
    "seatIndex": 1,
    "playerId": "player_234"
  }
}
```

### PlayerReadyChanged

```json
{
  "eventType": "PlayerReadyChanged",
  "roomId": "room_456",
  "stateVersion": 5,
  "payload": {
    "playerId": "player_234",
    "isReady": true
  }
}
```

### GameStarted

```json
{
  "eventType": "GameStarted",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 1,
  "payload": {
    "gameType": "SimpleHighCard",
    "players": ["player_123", "player_234"]
  }
}
```

### CardsDealt

Public event:

```json
{
  "eventType": "CardsDealt",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 2,
  "payload": {
    "handCounts": {
      "player_123": 1,
      "player_234": 1
    },
    "deckCount": 50
  }
}
```

Private event to player:

```json
{
  "eventType": "PrivateHandUpdated",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 2,
  "payload": {
    "playerId": "player_123",
    "cards": [
      { "rank": "Ace", "suit": "Spades" }
    ]
  }
}
```

### TurnChanged

```json
{
  "eventType": "TurnChanged",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 3,
  "payload": {
    "currentTurnPlayerId": "player_123",
    "turnNumber": 1
  }
}
```

### GameActionAccepted

```json
{
  "eventType": "GameActionAccepted",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 4,
  "payload": {
    "commandId": "cmd_01JZ0000000000000000000001",
    "playerId": "player_123",
    "actionType": "RevealCard"
  }
}
```

### GameActionRejected

Gửi riêng requester.

```json
{
  "eventType": "GameActionRejected",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 4,
  "payload": {
    "commandId": "cmd_01JZ0000000000000000000002",
    "errorCode": "CardNotInHand",
    "message": "The selected card is not in the player's hand."
  }
}
```

### GameStateUpdated

Gửi public hoặc private tùy payload.

```json
{
  "eventType": "GameStateUpdated",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 5,
  "payload": {
    "publicState": {
      "deckCount": 50,
      "handCounts": {
        "player_123": 1,
        "player_234": 1
      },
      "playedCards": []
    }
  }
}
```

### GameFinished

```json
{
  "eventType": "GameFinished",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 10,
  "payload": {
    "winnerPlayerIds": ["player_123"],
    "reason": "WinConditionMet",
    "publicResults": [
      { "playerId": "player_123", "rank": 1 },
      { "playerId": "player_234", "rank": 2 }
    ]
  }
}
```

### PlayerDisconnected

```json
{
  "eventType": "PlayerDisconnected",
  "roomId": "room_456",
  "stateVersion": 11,
  "payload": {
    "playerId": "player_234",
    "gracePeriodSeconds": 60
  }
}
```

### PlayerReconnected

```json
{
  "eventType": "PlayerReconnected",
  "roomId": "room_456",
  "stateVersion": 12,
  "payload": {
    "playerId": "player_234"
  }
}
```

### RoomClosed

```json
{
  "eventType": "RoomClosed",
  "roomId": "room_456",
  "stateVersion": 20,
  "payload": {
    "reason": "EmptyTimeout"
  }
}
```

## Reconnect Và State Snapshot

Snapshot response:

```json
{
  "eventType": "StateSnapshot",
  "roomId": "room_456",
  "gameSessionId": "game_789",
  "stateVersion": 12,
  "payload": {
    "room": {
      "roomId": "room_456",
      "roomCode": "ABCD12",
      "roomState": "Playing",
      "hostPlayerId": "player_123",
      "seats": [
        { "seatIndex": 0, "playerId": "player_123", "isReady": true, "isConnected": true },
        { "seatIndex": 1, "playerId": "player_234", "isReady": true, "isConnected": true }
      ]
    },
    "game": {
      "gameType": "SimpleHighCard",
      "status": "InProgress",
      "currentTurnPlayerId": "player_234",
      "publicState": {
        "deckCount": 50,
        "handCounts": {
          "player_123": 1,
          "player_234": 1
        },
        "playedCards": []
      },
      "privateState": {
        "hand": [
          { "rank": "King", "suit": "Hearts" }
        ]
      }
    }
  }
}
```

Rules:

- Snapshot là per-player view.
- Không gửi deck order.
- Không gửi hand của đối thủ.
- Nếu player không thuộc room, reject `RequestStateSync`.

## Public Và Private Payload

Public payload:

- Room state.
- Seat occupancy.
- Ready state.
- Player display names.
- Current turn player.
- Hand count của mọi player.
- Played/discard cards công khai.
- Game result công khai.

Private payload:

- Hand của chính player.
- Action options nếu chỉ player đó được biết.
- Reconnect/session token nếu cần.

Server-only:

- Deck order.
- Hidden hands của người khác.
- Random seed/internal RNG state.
- Command dedupe cache.
- Full action payload nếu chứa private data.

## Xử Lý Command Không Hợp Lệ

Command bị reject khi:

- Payload sai schema.
- Player chưa authenticated.
- Player không thuộc room.
- Room/session không tồn tại.
- Player không phải host nhưng gửi host-only command.
- Room state không cho command đó.
- Không phải lượt player.
- Action không hợp lệ theo game rule.
- `commandId` trùng và payload khác command đã xử lý.

Server response:

- Trả error response cho requester.
- Không broadcast public event nếu state không đổi.
- Có thể gửi `GameActionRejected` riêng requester.

## Tránh Hai Hành Động Gần Như Cùng Lúc

- Commands vào cùng room được serialize bằng per-room queue/lock.
- Command xử lý trước nếu accepted sẽ tăng `stateVersion`.
- Command xử lý sau được validate trên state mới.
- Nếu command sau không còn hợp lệ, reject với `ConflictError` hoặc `NotYourTurn`.

## Tránh Xử Lý Trùng Command

- Cache key: `(playerId, commandId)`.
- Nếu duplicate cùng payload, trả lại response cũ.
- Nếu duplicate khác payload, reject `DuplicateCommandPayloadMismatch`.
- TTL cache tối thiểu bằng thời gian reconnect grace period.

## Rate Limit Đề Xuất

- Room/lobby command: giới hạn thấp theo player.
- Game action: giới hạn theo turn và theo giây.
- Invalid command repeated: tăng penalty hoặc tạm ngắt connection.

## Error Codes Ban Đầu

- `InvalidPayload`
- `Unauthenticated`
- `Forbidden`
- `RoomNotFound`
- `RoomFull`
- `RoomClosed`
- `InvalidRoomState`
- `NotRoomHost`
- `NotRoomMember`
- `SeatUnavailable`
- `NotReady`
- `GameSessionNotFound`
- `InvalidGameState`
- `NotYourTurn`
- `UnsupportedActionType`
- `CardNotInHand`
- `DuplicateCommand`
- `DuplicateCommandPayloadMismatch`
- `StateOutOfDate`
- `RateLimited`
- `InternalError`
