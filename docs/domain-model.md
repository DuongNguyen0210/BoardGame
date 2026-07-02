# Domain Model

## Tổng Quan

Domain model được chia thành bốn vùng chính:

- Cards: mô hình bộ bài Tây 52 lá, không chứa luật game cụ thể.
- Players: identity và trạng thái kết nối tối thiểu.
- Rooms: aggregate quản lý phòng, seat, ready, host và room state.
- Games: game session, action, event, state, turn và result.

Domain phải thuần C# và có thể unit test độc lập với Unity, ASP.NET Core, SignalR và database.

## Entity

### Player

Đại diện một người chơi trong hệ thống.

Thuộc tính:

- `PlayerId`
- `DisplayName`
- `ConnectionStatus`
- `IsGuest`
- `LastSeenAt`

Trách nhiệm:

- Giữ identity và trạng thái online/offline.
- Không giữ game rule.
- Không tự quyết định room/game membership; room aggregate quản lý membership.

### Room

Aggregate root cho phòng game.

Thuộc tính:

- `RoomId`
- `RoomCode`
- `GameType`
- `HostPlayerId`
- `RoomState`
- `MinPlayers`
- `MaxPlayers`
- `Seats`
- `Members`
- `CurrentGameSessionId`
- `CreatedAt`
- `UpdatedAt`

Trách nhiệm:

- Thêm/rời player.
- Chọn/đổi seat.
- Set ready.
- Chuyển host.
- Kiểm tra điều kiện start.
- Quản lý room state machine.
- Liên kết game session hiện tại.

### Seat

Entity con trong `Room`.

Thuộc tính:

- `SeatIndex`
- `PlayerId?`
- `IsReady`
- `IsConnected`

Trách nhiệm:

- Đại diện một vị trí trong room.
- Bảo vệ không có hai player cùng một seat.
- Bảo vệ một player không ngồi nhiều seat trong cùng room.

### GameSession

Aggregate hoặc entity được quản lý bởi room/application.

Thuộc tính:

- `GameSessionId`
- `RoomId`
- `GameType`
- `Status`
- `StateVersion`
- `StartedAt`
- `FinishedAt?`
- `Players`
- `ServerGameState`
- `ActionHistory`
- `Result?`

Trách nhiệm:

- Giữ state của một ván.
- Áp dụng state transition qua game engine.
- Tăng `StateVersion`.
- Ghi action history.
- Không biết realtime transport.

### GameActionRecord

Record lịch sử action đã accepted/rejected nếu cần audit.

Thuộc tính:

- `CommandId`
- `GameSessionId`
- `PlayerId`
- `ActionType`
- `PayloadSummary`
- `StateVersionBefore`
- `StateVersionAfter?`
- `Accepted`
- `ErrorCode?`
- `CreatedAt`

Trách nhiệm:

- Hỗ trợ debug, replay đơn giản, idempotency và audit.
- Không chứa private payload dư thừa nếu không cần.

### MatchResult

Kết quả sau khi game kết thúc.

Thuộc tính:

- `GameSessionId`
- `WinnerPlayerIds`
- `PlayerResults`
- `FinishedReason`
- `FinishedAt`

Trách nhiệm:

- Đại diện kết quả authoritative do server/game engine tính.
- Là dữ liệu ứng viên để lưu lâu dài.

## Value Object

### PlayerId, RoomId, GameSessionId

Typed ID để tránh truyền nhầm string/guid giữa domain.

### RoomCode

Mã ngắn để join room.

Invariant:

- Không rỗng.
- Dễ nhập.
- Unique trong active room set.

### DisplayName

Tên hiển thị.

Invariant:

- Không rỗng.
- Giới hạn độ dài.
- Được sanitize ở application/API layer trước khi hiển thị.

### Card

Lá bài Tây.

Thuộc tính:

- `Suit`
- `Rank`

Invariant:

- Một `Card` là immutable.
- Không chứa logic so mạnh/yếu theo game.

### Deck

Collection các lá còn trong deck.

Trách nhiệm:

- Tạo từ card set.
- Shuffle qua `IShuffleService` hoặc `IRandomSource`.
- Draw một hoặc nhiều lá.
- Không quyết định luật chia bài đặc thù nếu luật đó thuộc game.

Invariant:

- Không có lá trùng.
- Draw làm giảm deck count.
- Một lá đã draw không còn trong deck.

### Hand

Collection bài của một player trong game session.

Trách nhiệm:

- Add/remove cards.
- Kiểm tra player có lá cụ thể.
- Không tự xác định combo hợp lệ nếu combo là luật game.

### CardCollection

Base/helper value object cho các tập bài như played cards, discard pile.

## Aggregate

### Room Aggregate

Aggregate root: `Room`.

Entity con:

- `Seat`.
- Member entry nếu cần tách khỏi seat.

Invariant:

- Một người chơi không được ngồi ở hai seat trong cùng một room.
- Một seat không được chứa hai người chơi.
- Room không vượt quá `MaxPlayers`.
- Host phải là member của room.
- Chỉ host mới được bắt đầu game.
- Chỉ bắt đầu game khi đủ người và tất cả seated players hợp lệ đã ready.
- Không đổi seat khi room đang `Starting` hoặc `Playing`.
- Khi host rời, host mới phải là member còn lại nếu có.

### GameSession Aggregate

Aggregate root: `GameSession`.

State con:

- `ServerGameState`.
- `Hands`.
- `Deck`.
- `TurnState`.
- `ActionHistory`.
- `Result`.

Invariant:

- Chỉ người đang có lượt mới được thực hiện turn action.
- Một lá bài không thể đồng thời nằm trong deck và trên tay player.
- Một lá bài không thể nằm trên tay hai player.
- Một command đã xử lý không được xử lý lại.
- `StateVersion` chỉ tăng khi state đổi thành công.
- `Winner` chỉ được set khi win condition được thỏa.
- Client không thể cung cấp deck order, dealt cards hoặc result.

## Enum

### Suit

- `Clubs`
- `Diamonds`
- `Hearts`
- `Spades`

Không có thứ tự mạnh/yếu mặc định trong card domain.

### Rank

- `Two`
- `Three`
- `Four`
- `Five`
- `Six`
- `Seven`
- `Eight`
- `Nine`
- `Ten`
- `Jack`
- `Queen`
- `King`
- `Ace`

Thứ tự enum chỉ là identity nội bộ. Game module quyết định thứ tự so sánh.

### ConnectionStatus

- `Online`
- `Disconnected`
- `Offline`

### RoomState

- `Waiting`
- `ReadyCheck`
- `Starting`
- `Playing`
- `Finished`
- `Closed`

### GameSessionStatus

- `Created`
- `Initializing`
- `Dealing`
- `InProgress`
- `Paused`
- `Finished`
- `Archived`

### CommandStatus

- `Accepted`
- `Rejected`
- `Duplicate`

## Interface

### Nên có ở giai đoạn đầu

`IGameDefinition`

- Cung cấp `GameType`, display name, min/max player, action types.
- Tạo engine/state ban đầu hoặc khai báo factory cần dùng.

`IGameEngine<TState>`

- `Initialize`.
- `ValidateAction`.
- `ApplyAction`.
- `CreatePlayerView`.
- `CheckFinished`.

`IGameAction`

- Contract chung cho action sau khi parse/validate payload.
- Có `ActionType`, `PlayerId`, metadata cần thiết.

`IGameEvent`

- Event domain phát sinh sau state transition.
- Không chứa transport concern.

`IDeckFactory`

- Tạo deck chuẩn 52 lá hoặc deck tùy game sau này.

`IShuffleService` hoặc `IRandomSource`

- Shuffle deterministic trong test.
- Random implementation nằm ở infrastructure.

`ITurnManager`

- Quản lý current player, next player, skip/eliminate nếu game cần.
- Có thể là class concrete nếu ban đầu chỉ có một implementation.

`ICommandDeduplicationStore`

- Theo dõi `commandId` đã xử lý trong room/session.
- Có thể in-memory ở giai đoạn đầu, cache sau này.

### Chưa nên tạo ở giai đoạn đầu

`IGameRule`

- Quá rộng và dễ mơ hồ. Nên dùng method cụ thể trong game engine hoặc service rõ nghĩa.

`IGameEventHandler`

- Chưa cần nếu event chỉ được application mapping/broadcast.

`IScoreCalculator`

- Chỉ tạo khi game đầu tiên có scoring thật.

`IGameStateValidator`

- Chỉ tách khi validation phức tạp hoặc cần dùng lại giữa nhiều action.

`ICardComparer` ở domain chung

- Card domain không có thứ tự mạnh/yếu chung. Nếu sample game cần, đặt trong `GameModules.SimpleHighCard`.

## Domain Service

### DeckFactory

Tạo deck 52 lá chuẩn.

### ShuffleService

Xáo deck qua random source.

### RoomStatePolicy

Kiểm tra transition room hợp lệ nếu logic state machine lớn hơn aggregate method.

### GameDefinitionRegistry

Map `GameType` sang `IGameDefinition`.

### TurnManager

Quản lý lượt chơi cho game turn-based.

### WinCondition

Kiểm tra kết thúc game trong từng game module.

### PlayerViewFactory

Tạo public/private view từ server state. Có thể nằm trong game engine nếu mỗi game có view khác nhau.

## Quan Hệ Giữa Các Đối Tượng

- `Player` tham gia `Room` qua `Member`/`Seat`.
- `Room` có nhiều `Seat`.
- `Room` có tối đa một `CurrentGameSession`.
- `GameSession` thuộc một `Room`.
- `GameSession` dùng `Deck` và nhiều `Hand`.
- `Hand` thuộc một `PlayerId` trong session.
- `GameAction` do một `PlayerId` gửi vào một `GameSession`.
- `GameAction` có thể tạo một hoặc nhiều `GameEvent`.
- `GameEvent` được application chuyển thành realtime event.
- `MatchResult` được tạo từ `GameSession` khi kết thúc.

## Invariant Cần Được Bảo Vệ

- Một người chơi không được ngồi ở hai seat trong cùng một room.
- Một seat không được chứa hai người chơi.
- Room không vượt quá số lượng player tối đa.
- Player không được ready nếu chưa ở seat hợp lệ.
- Chỉ chủ phòng mới được bắt đầu game.
- Chỉ bắt đầu game khi đủ người và tất cả người chơi hợp lệ đã ready.
- Room đang `Playing` không cho join như player thường ở giai đoạn đầu.
- Chỉ người đang có lượt mới được thực hiện turn action.
- Không thể đánh một lá không có trên tay.
- Một lá bài không thể đồng thời nằm trong deck và trên tay người chơi.
- Một command đã xử lý không được xử lý lại.
- State version không được giảm.
- Game result chỉ do game engine/server tạo.
- Private state của player này không xuất hiện trong player view của player khác.

## Game Mẫu Tối Giản

Game đề xuất: `SimpleHighCard`.

Luật:

- 2 player.
- Mỗi player được chia 1 lá.
- Cả hai player gửi `RevealCard` hoặc server auto reveal sau khi start tùy variant.
- Lá có rank cao hơn thắng.
- Nếu rank bằng nhau, dùng suit comparer của game module hoặc declare draw.

Mục đích test:

- Tạo room.
- Join.
- Ready.
- Start game.
- Shuffle/deal.
- Private hand.
- Perform action.
- State sync.
- Finish game.
- Play again.

Lưu ý: Thứ tự rank/suit của game mẫu nằm trong `SimpleHighCard`, không nằm trong card domain.
