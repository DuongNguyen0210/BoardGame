# Board Game Online Foundation Spec

## 1. Tổng Quan Dự Án

Repository hiện tại là một Unity project mới cho game 2D:

- Unity Editor: `6000.1.10f1`.
- Render pipeline: Universal Render Pipeline 2D.
- Package đáng chú ý: Unity Input System, Unity UI, Unity Test Framework, Unity Multiplayer Center, MCP for Unity.
- Chưa có script C#, backend, database, API, realtime hub, hoặc frontend web.
- Thư mục `docs` được tạo cho giai đoạn spec này.

Assumption chính: Unity client sẽ là frontend đầu tiên. Backend authoritative server sẽ được thêm sau khi spec được duyệt, ưu tiên ASP.NET Core vì cùng hệ sinh thái C# với Unity và phù hợp SignalR/WebSocket. Nếu muốn dùng Node.js, Go hoặc Unity dedicated server thì cần chốt lại trước khi implement.

## 2. Mục Tiêu

Tạo nền tảng chung cho nhiều game bài dùng bộ bài Tây 52 lá. Giai đoạn đầu không xây đầy đủ Tiến Lên, Phỏm, Poker hay Blackjack, mà xây khung xương có thể mở rộng:

- Server là nguồn dữ liệu chính xác duy nhất.
- Client chỉ gửi command và hiển thị state/event từ server.
- Luật game nằm trong game module/game engine, không nằm trong Controller, Hub hoặc UI.
- Card domain độc lập với luật game cụ thể.
- Có thể unit test game engine không cần Unity, network hoặc database.
- Dễ thêm game mới bằng cách thêm game definition/module thay vì sửa nhiều code nền tảng.

## 3. Phạm Vi Giai Đoạn Đầu

Giai đoạn đầu tạo foundation tối thiểu:

- User/guest identity cơ bản.
- Lobby tạo/tham gia/rời/tìm phòng.
- Room quản lý seat, ready, host, state machine.
- Card domain 52 lá độc lập luật game.
- Game engine abstraction và simple sample game.
- Realtime contract cho command, event, response và state snapshot.
- Persistence strategy ở mức thiết kế, chưa migration.
- Concurrency strategy theo room.
- Unity frontend skeleton ở mức khung.
- Unit/integration/E2E testing strategy.

## 4. Những Chức Năng Chưa Làm

Ngoài phạm vi giai đoạn đầu:

- Game bài hoàn chỉnh có luật phức tạp.
- Matchmaking nâng cao.
- Rank/rating.
- Tournament.
- Chat.
- Friend system.
- Spectator.
- Replay hoàn chỉnh.
- Anti-cheat nâng cao.
- Game server phân tán.
- Redis cluster.
- Microservices.
- Payment.
- Shop.
- Cosmetic.
- AI bot nâng cao.
- Voice chat.

## 5. Các Actor Trong Hệ Thống

- Guest player: truy cập nhanh, có `PlayerId`, display name tạm, reconnect token.
- Registered player: tài khoản đăng nhập sau này, vẫn dùng `PlayerId` ổn định.
- Room host: người tạo phòng hoặc được chuyển quyền host.
- Server: authoritative state owner, kiểm tra command, chạy game engine, broadcast event.
- Unity client: gửi command, render lobby/room/table, không quyết định kết quả game.
- Admin/operator: chưa làm UI ở giai đoạn đầu, chỉ là actor tương lai để quản trị phòng/session.

## 6. Các Use Case Chính

- Người chơi vào hệ thống bằng guest name hoặc đăng nhập cơ bản.
- Người chơi xem danh sách room.
- Người chơi tạo room theo game type.
- Người chơi nhập room code để tham gia.
- Người chơi chọn seat và ready.
- Host bắt đầu game khi đủ điều kiện.
- Server tạo game session, chia bài, gửi view riêng cho từng player.
- Player gửi action trong lượt.
- Server validate action, cập nhật state, phát event.
- Game kết thúc, server xác định winner/result.
- Player disconnect và reconnect nhận lại snapshot đúng quyền nhìn.
- Host rời room, server chuyển host.
- Player play again để tạo session mới trong cùng room.

## 7. Luồng Tạo Phòng

1. Client gửi `CreateRoom` với `commandId`, `playerId`, `gameType`, cấu hình min/max player.
2. Application service xác thực player đang online và chưa ở trạng thái xung đột.
3. Room service tạo `RoomId`, `RoomCode`, host là người tạo, state `Waiting`.
4. Server thêm host vào seat đầu tiên hoặc để host chọn seat, tùy quyết định UI.
5. Server trả response thành công cho requester.
6. Server broadcast `RoomCreated` hoặc cập nhật lobby bằng `RoomListUpdated`.

Invariant:

- `RoomCode` duy nhất trong các phòng đang mở.
- Host phải là member của room.
- Không tạo room với min/max ngoài range game definition cho phép.

## 8. Luồng Tham Gia Phòng

1. Client gửi `JoinRoom` bằng `roomCode` hoặc `roomId`.
2. Server kiểm tra room tồn tại, chưa `Closed`, chưa full, và không ở state cấm join.
3. Server thêm player vào members. Nếu room đang `Waiting`/`ReadyCheck`, player có thể chọn seat.
4. Server trả `JoinRoomResult` kèm room snapshot theo quyền player.
5. Server broadcast `PlayerJoined` cho người trong room và lobby update.

Policy đề xuất: giai đoạn đầu chỉ cho join khi room `Waiting` hoặc `ReadyCheck`. Spectator và join giữa ván để xem sẽ để sau.

## 9. Luồng Ready Và Bắt Đầu Game

1. Player đã ngồi vào seat gửi `SetReady(true)`.
2. Server kiểm tra player thuộc room, seat hợp lệ, room đang `Waiting` hoặc `ReadyCheck`.
3. Server cập nhật ready state và broadcast `PlayerReadyChanged`.
4. Khi đủ min player và tất cả seated players ready, room có thể chuyển `ReadyCheck`.
5. Host gửi `StartGame`.
6. Server kiểm tra host quyền hợp lệ, room đủ điều kiện, game definition tồn tại.
7. Room chuyển `Starting`, server tạo `GameSession`.
8. Game engine initialize, tạo/shuffle/deal, sinh event.
9. Room chuyển `Playing`, server gửi `GameStarted`, `CardsDealt`, `GameStateUpdated` theo player view.

## 10. Luồng Thực Hiện Một Lượt Chơi

1. Client đang tới lượt gửi `PerformGameAction` với `commandId`, `roomId`, `gameSessionId`, `stateVersion`, `actionType`, `payload`.
2. Server đưa command vào queue/lock của room.
3. Server kiểm tra idempotency bằng `commandId`.
4. Server kiểm tra player thuộc room, session đang active, player có quyền hành động, state version không quá cũ.
5. Game engine validate action bằng rule của game module.
6. Nếu invalid, server trả `GameActionRejected` cho requester, không đổi state.
7. Nếu valid, game engine apply action, tăng `stateVersion`, ghi `ActionHistory`, sinh domain events.
8. Server broadcast event public và gửi private state update đúng từng player.
9. Turn manager chuyển lượt hoặc win condition kết thúc game.

## 11. Luồng Reconnect

1. Client mất kết nối, server đánh dấu `PlayerConnectionStatus = Disconnected`, giữ seat/room trong grace period.
2. Server broadcast `PlayerDisconnected` cho room.
3. Client reconnect bằng token/session identity.
4. Server map connection mới với `PlayerId`.
5. Server đánh dấu online, broadcast `PlayerReconnected`.
6. Server gửi `StateSnapshot` riêng cho player:
   - Room snapshot.
   - Game session snapshot nếu đang chơi.
   - Private hand của chính player.
   - `stateVersion` mới nhất.
   - Command result cache nếu cần khôi phục command vừa gửi trước khi mất mạng.

Giai đoạn đầu lưu reconnect information trong memory. Sau này có thể đưa vào distributed cache nếu cần scale nhiều process.

## 12. Domain Model

Nhóm domain chính:

- Cards: `Suit`, `Rank`, `Card`, `Deck`, `Hand`, `CardCollection`, `DiscardPile`.
- Players: `PlayerId`, `Player`, `DisplayName`, `ConnectionStatus`.
- Rooms: `RoomId`, `RoomCode`, `Room`, `Seat`, `RoomState`, `ReadyState`.
- Games: `GameType`, `GameSession`, `GameState`, `GameAction`, `GameEvent`, `TurnState`, `ActionHistory`.
- Results: `MatchResult`, `PlayerResult`, `Score`.

Card domain không biết luật game. Ví dụ lá 2 mạnh nhất, A đứng trước 2, chất nào mạnh hơn đều thuộc game module qua `CardRankPolicy`/`CardComparer`.

## 13. Game Engine Architecture

Interface nên có ở giai đoạn đầu:

- `IGameDefinition`: metadata và factory cho một game type, gồm min/max player, initial deal rule, action schema.
- `IGameEngine`: xử lý initialize, validate/apply action, tạo player view, kiểm tra finish.
- `IGameAction`: command nội bộ của game, được tạo từ realtime command sau khi server validate quyền cơ bản.
- `IGameEvent`: event phát sinh từ state transition.
- `IGameState`: marker/contract cho state của từng game module.
- `IDeckFactory`: tạo deck chuẩn hoặc deck tùy game nếu sau này cần.
- `IRandomSource` hoặc `IShuffleService`: shuffle có thể test được, không phụ thuộc `System.Random` trực tiếp.
- `ITurnManager`: quản lý lượt nếu nhiều game dùng lại được.
- `IWinCondition`: tách điều kiện thắng nếu game đơn giản hoặc có nhiều variant.

Interface chưa nên tạo ngay:

- `IGameRule` dạng quá rộng: dễ thành abstraction mơ hồ. Giai đoạn đầu đặt rule trong `IGameEngine` hoặc các service cụ thể của game mẫu.
- `IGameEventHandler`: chưa cần khi event chỉ broadcast và ghi history. Thêm khi có side effect phức tạp.
- `IScoreCalculator`: chỉ tạo khi sample game hoặc game đầu tiên thật sự có scoring.
- `ICardComparer`: chỉ tạo khi game mẫu cần so bài. Với sample "High Card" thì nên tạo `ICardComparer` trong game module, không đặt vào card domain.
- `IGameStateValidator`: chưa cần tách riêng nếu validation còn nhỏ. Tách khi validate phình ra hoặc cần test độc lập.

Quan hệ:

- Application service nhận command từ API/Hub.
- Application service gọi room service để kiểm tra quyền và state room.
- Room service gọi game engine của `GameSession`.
- Game engine chỉ biết domain model và game definition, không biết SignalR/Unity/database.
- Realtime adapter chuyển game event/state view thành DTO gửi client.

## 14. Room State Machine

State đề xuất:

- `Waiting`: room mở, player có thể join/leave/select seat.
- `ReadyCheck`: đủ min player, đang chờ ready hoặc đã đủ ready chờ host start.
- `Starting`: server đang tạo game session, lock thay đổi seat/ready.
- `Playing`: game session active.
- `Finished`: session kết thúc, hiển thị kết quả, có thể play again.
- `Closed`: room đóng, không nhận command thường.

Điều kiện chuyển:

- `Waiting -> ReadyCheck`: đủ min seated players.
- `ReadyCheck -> Waiting`: số player hợp lệ dưới min hoặc người chơi unready làm mất điều kiện.
- `ReadyCheck -> Starting`: host gửi start và tất cả seated players ready.
- `Starting -> Playing`: game session initialize thành công.
- `Starting -> ReadyCheck`: initialize thất bại có thể phục hồi.
- `Playing -> Finished`: game engine báo finish.
- `Finished -> ReadyCheck`: đủ player chọn play again.
- `Waiting/ReadyCheck/Finished -> Closed`: host đóng room hoặc room empty timeout.
- `Playing -> Closed`: lỗi nghiêm trọng hoặc tất cả player rời/disconnect quá timeout.

## 15. Game State Model

Ba lớp state:

- Public state: mọi player trong room được biết. Ví dụ turn hiện tại, số bài trên tay mỗi người, bài đã đánh, điểm public, trạng thái ready.
- Private state: chỉ một player được biết. Ví dụ lá bài trên tay của chính player.
- Server state: chỉ server biết. Ví dụ thứ tự deck còn lại, seed/random state, hidden action metadata, command dedupe cache.

Cách tạo player view:

1. Game engine giữ `ServerGameState` đầy đủ trong memory.
2. `CreatePlayerView(playerId)` lọc state theo quyền:
   - Hand của player: gửi full cards.
   - Hand người khác: gửi count hoặc public cards nếu luật cho phép.
   - Deck: gửi count, không gửi thứ tự.
   - Discard/played cards: gửi public cards.
3. Realtime layer chỉ gửi `PlayerGameView`, không gửi `ServerGameState`.
4. Integration test phải kiểm tra payload của player A không chứa private cards của player B.

## 16. Realtime Protocol

Cơ chế đề xuất:

- Backend ASP.NET Core SignalR cho realtime nếu chốt stack .NET.
- Nếu backend khác, giữ contract command/event như tài liệu `realtime-contract.md` và thay transport sau.

Phân biệt:

- Command: ý định từ client lên server, có `commandId`.
- Request/response: kết quả trực tiếp cho requester, thành công hoặc lỗi.
- Event: server broadcast việc đã xảy ra sau khi state đổi.
- State snapshot: ảnh chụp state hiện tại cho initial load/reconnect/resync.

Command invalid:

- Không đổi game state.
- Trả error có `errorCode`, `message`, `stateVersion`.
- Với game action, có thể gửi `GameActionRejected` chỉ cho requester.

Chống race/trùng:

- Mỗi room xử lý tuần tự.
- `commandId` unique theo player/session.
- Dedupe cache lưu kết quả command đã xử lý trong một khoảng thời gian.
- `stateVersion` tăng sau mỗi accepted action.

## 17. Persistence Strategy

Giai đoạn đầu:

- In memory: room active, game session active, deck order, command queue, ready state, connection mapping.
- Long-lived DB sau này: user, registered profile, match result, player result, rating/rank.
- Optional DB/event store sau này: game action history, completed game session, replay data.
- Cache sau này: reconnect token, room lookup, online presence, command dedupe khi scale nhiều process.

Không triển khai microservices. Modular monolith là lựa chọn mặc định để giữ logic trong một codebase nhưng có ranh giới module rõ.

## 18. Concurrency Strategy

Chiến lược đề xuất giai đoạn đầu: mỗi room xử lý command tuần tự bằng lock hoặc per-room queue.

Ưu điểm:

- Dễ hiểu, ít race condition.
- Phù hợp game turn-based.
- Không cần distributed transaction.
- Unit/integration test đơn giản.

Nhược điểm:

- Một room có action chậm sẽ chặn command khác trong cùng room.
- Nếu chạy nhiều server process cần routing sticky theo room hoặc distributed actor/queue.
- Cần cẩn thận không giữ lock khi gọi I/O chậm.

Quy tắc:

- Validate và mutate room/game state trong critical section.
- Broadcast có thể thực hiện sau khi state đã được commit trong memory.
- Mỗi mutation tăng `stateVersion`.
- Command đến sai lượt bị reject, không chờ lượt.

## 19. Security Considerations

- Không tin dữ liệu từ client.
- Kiểm tra player có thực sự thuộc room.
- Kiểm tra player có quyền command: host-only, seated-only, current-turn-only.
- Kiểm tra command phù hợp game state hiện tại.
- Không gửi private hand của người khác.
- Không dùng dữ liệu client để quyết định shuffle/deal/winner.
- Không cho client tự gửi kết quả thắng thua.
- Không cho client tự đặt bài trên tay.
- Rate limit command theo player/connection.
- Validate payload size và schema.
- Reconnect token phải khó đoán và có TTL.
- Log command invalid để điều tra abuse.

## 20. Error-Handling Strategy

Nhóm lỗi:

- `ValidationError`: payload thiếu/sai.
- `AuthenticationError`: player chưa xác thực hoặc token sai.
- `AuthorizationError`: không thuộc room hoặc không có quyền.
- `ConflictError`: sai room/game state, sai lượt, state version cũ.
- `NotFoundError`: room/session/player không tồn tại.
- `DuplicateCommand`: command đã xử lý.
- `ServerError`: lỗi không mong muốn.

Quy tắc:

- Lỗi command không hợp lệ không làm đổi state.
- Response lỗi luôn chứa `commandId`, `errorCode`, `message`, `stateVersion` nếu có.
- Không trả chi tiết server state bí mật trong lỗi.
- Lỗi server khi `Starting` nên đưa room về `ReadyCheck` nếu có thể phục hồi.

## 21. Testing Strategy

Unit test:

- Deck có đúng 52 lá.
- Không có lá bài trùng.
- Shuffle không làm mất bài.
- Deal chuyển bài đúng vị trí.
- Room không vượt quá số lượng player.
- Player không được ready khi chưa hợp lệ.
- Không thể bắt đầu game khi chưa đủ điều kiện.
- Không thể hành động khi chưa tới lượt.
- Không thể đánh một lá không có trên tay.
- Command trùng không được xử lý hai lần.
- Game xác định đúng người thắng.

Integration test:

- Tạo room.
- Hai player tham gia.
- Ready.
- Start game.
- Thực hiện game action.
- Nhận event realtime.
- Kết thúc game.
- Reconnect và nhận đúng state.
- Người ngoài room không được gửi command vào room.

End-to-end test:

- Hai Unity client hoặc hai test client cùng tham gia một phòng.
- Chơi hoàn chỉnh game mẫu.
- Một client mất kết nối rồi reconnect.
- Client không nhìn thấy bài bí mật của đối thủ.

## 22. Definition of Done Cho Giai Đoạn Đầu

- Domain card, player, room và game engine sample có unit test.
- Có simple sample game chạy end-to-end qua application service.
- Server validate mọi command trước khi mutate state.
- Realtime contract có commandId, stateVersion, reconnect snapshot.
- Unity client skeleton có guest entry, lobby, room, game table.
- Không có luật game quan trọng trong UI/Hub/Controller.
- Private state không bị gửi nhầm cho player khác.
- Duplicate command không bị xử lý hai lần.
- Có tài liệu kiến trúc, domain model, realtime contract, roadmap.

## Quyết Định Kiến Trúc Cần Xác Nhận

### Backend stack

- Vấn đề: repository hiện chưa có backend.
- Lựa chọn:
  - ASP.NET Core + SignalR.
  - Node.js + WebSocket.
  - Unity dedicated server.
- Ưu/nhược điểm:
  - ASP.NET Core: cùng C#, SignalR tốt, unit test tốt; cần thêm project backend.
  - Node.js: phổ biến realtime; khác ngôn ngữ với Unity.
  - Unity dedicated server: dùng chung Unity runtime; dễ kéo logic vào engine/UI và khó test headless nếu không kỷ luật.
- Đề xuất: ASP.NET Core modular monolith.
- Lý do: phù hợp yêu cầu SOLID, DI, unit test game engine, SignalR realtime và chia module rõ.

### Realtime transport

- Vấn đề: cần realtime cho room và game.
- Lựa chọn: SignalR, raw WebSocket, polling.
- Đề xuất: SignalR nếu dùng ASP.NET Core.
- Lý do: group theo room, reconnect support, typed hub/client, giảm boilerplate.

### Persistence giai đoạn đầu

- Vấn đề: lưu bao nhiêu state ngay từ đầu.
- Lựa chọn: in-memory trước, database ngay, event sourcing.
- Đề xuất: in-memory active session + repository abstraction tối thiểu cho future persistence.
- Lý do: giảm scope, vẫn không khóa đường thêm DB sau.
