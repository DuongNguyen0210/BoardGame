# Development Roadmap

## Tổng Quan

Roadmap này chia foundation thành các phase nhỏ, ưu tiên domain model, game engine, unit test, application service, realtime, frontend và cuối cùng mới persistence nâng cao.

Repository hiện tại chỉ có Unity project mới. Các module backend dưới đây là đề xuất sau khi spec được duyệt, chưa được tạo trong task này.

## Phase 0: Phân Tích Repository Và Chốt Kiến Trúc

### Mục tiêu

Xác nhận hiện trạng công nghệ, assumptions và kiến trúc mục tiêu trước khi code.

### Task 0.1: Ghi nhận tech stack hiện tại

- Loại: documentation.
- File/module dự kiến: `docs/*`.
- Dependency: none.
- Tiêu chí hoàn thành:
  - Tech stack hiện tại được ghi rõ.
  - Assumption backend/realtime/persistence được đánh dấu.
  - Không sửa code nghiệp vụ.
- Test cần thực hiện:
  - Review tài liệu.
- Chưa làm:
  - Không tạo backend project.
  - Không thêm package.

### Task 0.2: Chốt backend stack

- Loại: architecture decision.
- File/module dự kiến: `docs/architecture.md`, có thể thêm ADR sau.
- Dependency: Task 0.1.
- Tiêu chí hoàn thành:
  - Quyết định ASP.NET Core + SignalR hoặc stack khác được duyệt.
  - Dependency direction được duyệt.
- Test cần thực hiện:
  - Review quyết định với owner.
- Chưa làm:
  - Không implement server.

## Phase 1: Card Domain

### Mục tiêu

Tạo card domain 52 lá độc lập với luật game cụ thể.

### Task 1.1: Tạo card value objects

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Cards/Card.cs`
  - `Server.Domain/Cards/Suit.cs`
  - `Server.Domain/Cards/Rank.cs`
- Dependency: Phase 0 approved.
- Tiêu chí hoàn thành:
  - `Card` immutable.
  - `Suit` và `Rank` chỉ định danh, không chứa rule mạnh/yếu.
  - Không phụ thuộc framework.
- Test cần thực hiện:
  - Unit test tạo card hợp lệ.
- Chưa làm:
  - Không tạo comparer game-specific.

### Task 1.2: Tạo deck factory chuẩn 52 lá

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Cards/Deck.cs`
  - `Server.Domain/Cards/DeckFactory.cs`
  - `Server.Tests/Domain/Cards/DeckTests.cs`
- Dependency: Task 1.1.
- Tiêu chí hoàn thành:
  - Deck có đúng 52 lá.
  - Không có lá trùng.
  - Draw giảm deck count.
- Test cần thực hiện:
  - Deck has 52 cards.
  - Deck has no duplicates.
  - Draw removes cards from deck.
- Chưa làm:
  - Không implement discard pile nếu sample chưa cần.

### Task 1.3: Tạo shuffle service testable

- Loại: domain/infrastructure boundary.
- File/module dự kiến:
  - `Server.Domain/Cards/IShuffleService.cs`
  - `Server.Domain/Common/IRandomSource.cs`
  - `Server.Infrastructure/Random/SystemRandomSource.cs`
  - `Server.Tests/Domain/Cards/ShuffleTests.cs`
- Dependency: Task 1.2.
- Tiêu chí hoàn thành:
  - Shuffle không mất bài.
  - Shuffle không tạo duplicate.
  - Test có deterministic random source.
- Test cần thực hiện:
  - Shuffle preserves card set.
- Chưa làm:
  - Không dùng cryptographic randomness nếu chưa cần anti-cheat nâng cao.

### Task 1.4: Tạo hand/card collection

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Cards/Hand.cs`
  - `Server.Domain/Cards/CardCollection.cs`
  - `Server.Tests/Domain/Cards/HandTests.cs`
- Dependency: Task 1.1.
- Tiêu chí hoàn thành:
  - Add/remove card rõ ràng.
  - Không remove lá không tồn tại mà không báo lỗi.
  - Không chứa combo validation game-specific.
- Test cần thực hiện:
  - Add card.
  - Remove existing card.
  - Reject removing missing card.
- Chưa làm:
  - Không tạo UI hand.

## Phase 2: Player Và Room Domain

### Mục tiêu

Tạo room aggregate và state machine cơ bản.

### Task 2.1: Tạo player identity model

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Players/Player.cs`
  - `Server.Domain/Players/PlayerId.cs`
  - `Server.Domain/Players/ConnectionStatus.cs`
- Dependency: Phase 0 approved.
- Tiêu chí hoàn thành:
  - Có player id, display name, guest flag, connection status.
  - Không có profile phức tạp.
- Test cần thực hiện:
  - Validate display name.
  - Change connection status.
- Chưa làm:
  - Không làm login database.

### Task 2.2: Tạo room và seat model

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Rooms/Room.cs`
  - `Server.Domain/Rooms/Seat.cs`
  - `Server.Domain/Rooms/RoomState.cs`
  - `Server.Tests/Domain/Rooms/RoomTests.cs`
- Dependency: Task 2.1.
- Tiêu chí hoàn thành:
  - Join/leave/select seat hoạt động.
  - Không vượt max players.
  - Một player không ngồi hai seat.
  - Một seat không chứa hai player.
- Test cần thực hiện:
  - Room capacity.
  - Seat conflict.
  - Player double-seat rejection.
- Chưa làm:
  - Không realtime.

### Task 2.3: Tạo ready state và start condition

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Rooms/Room.cs`
  - `Server.Tests/Domain/Rooms/ReadyTests.cs`
- Dependency: Task 2.2.
- Tiêu chí hoàn thành:
  - Player không ready khi chưa có seat.
  - Chỉ host start.
  - Không start khi thiếu player hoặc chưa ready.
- Test cần thực hiện:
  - Cannot ready without seat.
  - Cannot start if not host.
  - Cannot start when not all ready.
- Chưa làm:
  - Không tạo game session thật.

### Task 2.4: Tạo room state machine

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Rooms/RoomStateMachine.cs` hoặc method trong `Room`
  - `Server.Tests/Domain/Rooms/RoomStateMachineTests.cs`
- Dependency: Task 2.3.
- Tiêu chí hoàn thành:
  - Transition hợp lệ theo spec.
  - Reject transition không hợp lệ.
  - Host transfer khi host rời.
- Test cần thực hiện:
  - Waiting -> ReadyCheck.
  - ReadyCheck -> Starting.
  - Playing -> Finished.
  - Host transfer.
- Chưa làm:
  - Không persist room state.

## Phase 3: Game Engine Abstraction

### Mục tiêu

Tạo contract engine dùng chung cho nhiều game bài.

### Task 3.1: Tạo game definition registry

- Loại: domain/application boundary.
- File/module dự kiến:
  - `Server.Domain/Games/IGameDefinition.cs`
  - `Server.Application/Games/GameDefinitionRegistry.cs`
  - `Server.Tests/Application/Games/GameDefinitionRegistryTests.cs`
- Dependency: Phase 1, Phase 2.
- Tiêu chí hoàn thành:
  - Register game definition.
  - Resolve game by `GameType`.
  - Reject unknown game type.
- Test cần thực hiện:
  - Resolve known game.
  - Unknown game rejected.
- Chưa làm:
  - Không implement game phức tạp.

### Task 3.2: Tạo game session model

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Games/GameSession.cs`
  - `Server.Domain/Games/GameSessionStatus.cs`
  - `Server.Domain/Games/GameResult.cs`
  - `Server.Tests/Domain/Games/GameSessionTests.cs`
- Dependency: Task 3.1.
- Tiêu chí hoàn thành:
  - Có session id, room id, game type, status, state version.
  - State version tăng khi accepted transition.
  - Action history append được.
- Test cần thực hiện:
  - Initial version.
  - Version increments after accepted action.
  - Action history records accepted action.
- Chưa làm:
  - Không lưu database.

### Task 3.3: Tạo game action/event contracts

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Games/IGameAction.cs`
  - `Server.Domain/Games/IGameEvent.cs`
  - `Server.Domain/Games/GameActionResult.cs`
- Dependency: Task 3.2.
- Tiêu chí hoàn thành:
  - Action/result/event contract đủ cho sample game.
  - Không phụ thuộc SignalR DTO.
- Test cần thực hiện:
  - Contract usage in fake game test.
- Chưa làm:
  - Không tạo event handler abstraction nếu chưa cần.

### Task 3.4: Tạo game engine interface và base flow

- Loại: domain.
- File/module dự kiến:
  - `Server.Domain/Games/IGameEngine.cs`
  - `Server.Domain/Games/PlayerGameView.cs`
  - `Server.Tests/Domain/Games/FakeGameEngineTests.cs`
- Dependency: Task 3.3.
- Tiêu chí hoàn thành:
  - Initialize game.
  - Validate/apply action.
  - Create player view.
  - Check finished.
- Test cần thực hiện:
  - Invalid action does not mutate.
  - Player view filters private state.
- Chưa làm:
  - Không implement real game rules ngoài fake/sample.

### Task 3.5: Tạo command dedupe và state version policy

- Loại: application/domain support.
- File/module dự kiến:
  - `Server.Application/Commands/ICommandDeduplicationStore.cs`
  - `Server.Infrastructure/Commands/InMemoryCommandDeduplicationStore.cs`
  - `Server.Tests/Application/Commands/CommandDeduplicationTests.cs`
- Dependency: Task 3.2.
- Tiêu chí hoàn thành:
  - Duplicate command trả result cũ.
  - Duplicate cùng id khác payload bị reject.
  - State version conflict có error rõ.
- Test cần thực hiện:
  - Duplicate command not processed twice.
  - Payload mismatch rejected.
- Chưa làm:
  - Không distributed cache.

## Phase 4: Game Mẫu Tối Giản

### Mục tiêu

Implement `SimpleHighCard` để kiểm tra kiến trúc.

### Task 4.1: Tạo SimpleHighCard definition

- Loại: game module.
- File/module dự kiến:
  - `Server.GameModules/SimpleHighCard/SimpleHighCardDefinition.cs`
  - `Server.Tests/GameModules/SimpleHighCard/SimpleHighCardDefinitionTests.cs`
- Dependency: Phase 3.
- Tiêu chí hoàn thành:
  - Game type là `SimpleHighCard`.
  - Min/max player = 2.
  - Supported action type rõ.
- Test cần thực hiện:
  - Definition metadata valid.
- Chưa làm:
  - Không UI.

### Task 4.2: Implement deal và private view

- Loại: game module.
- File/module dự kiến:
  - `Server.GameModules/SimpleHighCard/SimpleHighCardEngine.cs`
  - `Server.GameModules/SimpleHighCard/SimpleHighCardState.cs`
  - `Server.Tests/GameModules/SimpleHighCard/SimpleHighCardDealTests.cs`
- Dependency: Task 4.1, Phase 1.
- Tiêu chí hoàn thành:
  - Mỗi player nhận một lá.
  - Public state chỉ có hand count/deck count.
  - Private view chỉ có hand của player đó.
- Test cần thực hiện:
  - Cards dealt to both players.
  - Opponent private card not visible.
- Chưa làm:
  - Không animation/card art.

### Task 4.3: Implement reveal/action flow

- Loại: game module.
- File/module dự kiến:
  - `Server.GameModules/SimpleHighCard/SimpleHighCardActions.cs`
  - `Server.Tests/GameModules/SimpleHighCard/SimpleHighCardActionTests.cs`
- Dependency: Task 4.2.
- Tiêu chí hoàn thành:
  - Chỉ current player hoặc required player được reveal theo rule đã chọn.
  - Không reveal lá không có trên tay.
  - Accepted action tạo event.
- Test cần thực hiện:
  - Cannot act when not turn.
  - Cannot reveal missing card.
  - Accepted reveal updates state.
- Chưa làm:
  - Không support nhiều variant.

### Task 4.4: Implement win condition

- Loại: game module.
- File/module dự kiến:
  - `Server.GameModules/SimpleHighCard/SimpleHighCardComparer.cs`
  - `Server.GameModules/SimpleHighCard/SimpleHighCardWinCondition.cs`
  - `Server.Tests/GameModules/SimpleHighCard/SimpleHighCardResultTests.cs`
- Dependency: Task 4.3.
- Tiêu chí hoàn thành:
  - Game xác định winner đúng.
  - Tie policy rõ: draw hoặc suit tiebreaker.
  - Result do server tạo.
- Test cần thực hiện:
  - Higher rank wins.
  - Tie handled.
  - Winner not accepted from client payload.
- Chưa làm:
  - Không rating/rank.

## Phase 5: Application Services

### Mục tiêu

Kết nối use case room/game bằng service layer, chưa cần realtime thật.

### Task 5.1: Create/join/leave room handlers

- Loại: application service.
- File/module dự kiến:
  - `Server.Application/Rooms/CreateRoomHandler.cs`
  - `Server.Application/Rooms/JoinRoomHandler.cs`
  - `Server.Application/Rooms/LeaveRoomHandler.cs`
  - `Server.Tests/Application/Rooms/RoomHandlerTests.cs`
- Dependency: Phase 2.
- Tiêu chí hoàn thành:
  - Create room tạo host.
  - Join room kiểm tra capacity/state.
  - Leave room chuyển host hoặc close nếu empty.
- Test cần thực hiện:
  - Create room.
  - Join full room rejected.
  - Host leave transfers host.
- Chưa làm:
  - Không SignalR.

### Task 5.2: Ready/start game handlers

- Loại: application service.
- File/module dự kiến:
  - `Server.Application/Rooms/SetReadyHandler.cs`
  - `Server.Application/Rooms/StartGameHandler.cs`
  - `Server.Tests/Application/Rooms/StartGameHandlerTests.cs`
- Dependency: Phase 3, Task 5.1.
- Tiêu chí hoàn thành:
  - Ready cập nhật room.
  - Start tạo game session qua game definition.
  - Invalid start không mutate state.
- Test cần thực hiện:
  - Cannot start before ready.
  - Start creates game session.
- Chưa làm:
  - Không broadcast thật.

### Task 5.3: Perform game action handler

- Loại: application service.
- File/module dự kiến:
  - `Server.Application/Games/PerformGameActionHandler.cs`
  - `Server.Tests/Application/Games/PerformGameActionHandlerTests.cs`
- Dependency: Phase 4, Task 3.5.
- Tiêu chí hoàn thành:
  - Kiểm tra membership/session/state version.
  - Serialize xử lý theo room.
  - Duplicate command không apply hai lần.
- Test cần thực hiện:
  - Outsider rejected.
  - Not-your-turn rejected.
  - Duplicate command returns cached result.
- Chưa làm:
  - Không transport DTO.

### Task 5.4: Reconnect handler

- Loại: application service.
- File/module dự kiến:
  - `Server.Application/Players/ReconnectHandler.cs`
  - `Server.Application/Games/StateSnapshotService.cs`
  - `Server.Tests/Application/Players/ReconnectTests.cs`
- Dependency: Task 5.1, Task 5.3.
- Tiêu chí hoàn thành:
  - Map connection mới với player.
  - Trả room/game snapshot theo player view.
  - Không leak private cards.
- Test cần thực hiện:
  - Reconnect receives correct state.
  - Opponent hand hidden.
- Chưa làm:
  - Không persistent reconnect token.

## Phase 6: Realtime Layer

### Mục tiêu

Expose command/event qua SignalR hoặc WebSocket.

### Task 6.1: Tạo realtime DTO và hub skeleton

- Loại: realtime.
- File/module dự kiến:
  - `Server.API/Hubs/GameHub.cs`
  - `Server.API/Realtime/Commands/*.cs`
  - `Server.API/Realtime/Events/*.cs`
- Dependency: Phase 5.
- Tiêu chí hoàn thành:
  - Command envelope có `commandId`.
  - Hub chuyển command tới application service.
  - Không có game rule trong hub.
- Test cần thực hiện:
  - DTO serialization.
  - Hub calls handler in integration test.
- Chưa làm:
  - Không frontend Unity connect đầy đủ.

### Task 6.2: Implement room groups và event broadcasting

- Loại: realtime.
- File/module dự kiến:
  - `Server.Infrastructure/Realtime/SignalRRealtimeNotifier.cs`
  - `Server.Tests/Integration/Realtime/RoomBroadcastTests.cs`
- Dependency: Task 6.1.
- Tiêu chí hoàn thành:
  - Player join SignalR group theo room.
  - Public event broadcast đúng room.
  - Private event gửi đúng player.
- Test cần thực hiện:
  - Event received by room members.
  - Outsider does not receive room private event.
- Chưa làm:
  - Không scale-out SignalR backplane.

### Task 6.3: Implement state sync và reconnect over realtime

- Loại: realtime.
- File/module dự kiến:
  - `Server.API/Hubs/GameHub.cs`
  - `Server.Application/Games/StateSnapshotService.cs`
  - `Server.Tests/Integration/Realtime/ReconnectIntegrationTests.cs`
- Dependency: Task 6.2, Task 5.4.
- Tiêu chí hoàn thành:
  - Client reconnect nhận snapshot.
  - RequestStateSync hoạt động.
  - Snapshot không leak private state.
- Test cần thực hiện:
  - Reconnect and receive state.
  - State sync after stale version.
- Chưa làm:
  - Không distributed presence.

## Phase 7: Frontend Skeleton

### Mục tiêu

Unity client hiển thị state từ server và gửi command cơ bản.

### Task 7.1: Guest entry và realtime client adapter

- Loại: frontend.
- File/module dự kiến:
  - `Assets/Scripts/Client/Auth/GuestEntryController.cs`
  - `Assets/Scripts/Client/Realtime/GameRealtimeClient.cs`
- Dependency: Phase 6 contract stable.
- Tiêu chí hoàn thành:
  - Nhập display name.
  - Tạo guest player/session.
  - Kết nối realtime server.
- Test cần thực hiện:
  - Manual play mode test.
  - Connection error state.
- Chưa làm:
  - Không registered login.

### Task 7.2: Lobby page

- Loại: frontend.
- File/module dự kiến:
  - `Assets/Scripts/Client/Lobby/LobbyView.cs`
  - `Assets/Scripts/Client/Lobby/RoomListItemView.cs`
- Dependency: Task 7.1, Phase 6.
- Tiêu chí hoàn thành:
  - Hiển thị danh sách phòng.
  - Create room dialog.
  - Join room dialog theo code.
- Test cần thực hiện:
  - Manual two-client lobby test.
- Chưa làm:
  - Không matchmaking.

### Task 7.3: Waiting room page

- Loại: frontend.
- File/module dự kiến:
  - `Assets/Scripts/Client/Room/WaitingRoomView.cs`
  - `Assets/Scripts/Client/Room/SeatView.cs`
- Dependency: Task 7.2.
- Tiêu chí hoàn thành:
  - Hiển thị seats, ready, host.
  - Select seat.
  - Set ready.
  - Host start game button theo state server.
- Test cần thực hiện:
  - Manual ready/start flow.
- Chưa làm:
  - Không chat.

### Task 7.4: Game table page

- Loại: frontend.
- File/module dự kiến:
  - `Assets/Scripts/Client/GameTable/GameTableView.cs`
  - `Assets/Scripts/Client/GameTable/PlayerSeatView.cs`
  - `Assets/Scripts/Client/GameTable/PlayerHandView.cs`
  - `Assets/Scripts/Client/GameTable/TurnIndicatorView.cs`
- Dependency: Task 7.3, Phase 4, Phase 6.
- Tiêu chí hoàn thành:
  - Hiển thị player seats.
  - Hiển thị hand của mình.
  - Hiển thị hand count của đối thủ.
  - Gửi PerformGameAction.
  - Hiển thị turn indicator và notification.
- Test cần thực hiện:
  - Manual complete SimpleHighCard game.
  - Verify opponent private card hidden.
- Chưa làm:
  - Không animation nâng cao.

### Task 7.5: Reconnect screen

- Loại: frontend.
- File/module dự kiến:
  - `Assets/Scripts/Client/Connection/ReconnectView.cs`
  - `Assets/Scripts/Client/State/ClientStateStore.cs`
- Dependency: Task 7.4, Task 6.3.
- Tiêu chí hoàn thành:
  - Hiển thị mất kết nối.
  - Tự reconnect hoặc nút retry.
  - Nhận StateSnapshot và render lại.
- Test cần thực hiện:
  - Manual disconnect/reconnect.
- Chưa làm:
  - Không offline gameplay.

## Phase 8: Integration Testing

### Mục tiêu

Chứng minh foundation hoạt động qua nhiều client/test client.

### Task 8.1: Test create/join/ready/start

- Loại: integration test.
- File/module dự kiến:
  - `Server.Tests/Integration/RoomFlowTests.cs`
- Dependency: Phase 6.
- Tiêu chí hoàn thành:
  - Hai player tạo/join/ready/start được.
  - Event realtime đúng thứ tự đủ dùng.
- Test cần thực hiện:
  - Full room setup flow.
- Chưa làm:
  - Không E2E Unity automation nếu chưa có harness.

### Task 8.2: Test complete SimpleHighCard game

- Loại: integration test.
- File/module dự kiến:
  - `Server.Tests/Integration/SimpleHighCardFlowTests.cs`
- Dependency: Phase 4, Phase 6.
- Tiêu chí hoàn thành:
  - Start game.
  - Deal.
  - Perform actions.
  - GameFinished.
  - PlayAgain về ReadyCheck.
- Test cần thực hiện:
  - Complete sample game.
- Chưa làm:
  - Không complex game.

### Task 8.3: Test reconnect và privacy

- Loại: integration/E2E.
- File/module dự kiến:
  - `Server.Tests/Integration/ReconnectPrivacyTests.cs`
  - Unity manual test checklist.
- Dependency: Task 6.3, Task 7.5.
- Tiêu chí hoàn thành:
  - Reconnect nhận state đúng.
  - Player không nhìn thấy bài bí mật của đối thủ.
- Test cần thực hiện:
  - Reconnect snapshot privacy.
  - Two-client manual test.
- Chưa làm:
  - Không persistent reconnect across server restart.

### Task 8.4: Test invalid/duplicate/concurrent commands

- Loại: integration test.
- File/module dự kiến:
  - `Server.Tests/Integration/CommandConsistencyTests.cs`
- Dependency: Task 5.3, Phase 6.
- Tiêu chí hoàn thành:
  - Người ngoài room không gửi command vào room.
  - Command sai lượt bị reject.
  - Command trùng không xử lý hai lần.
  - Hai command gần đồng thời được serialize.
- Test cần thực hiện:
  - Outsider rejected.
  - Duplicate command deduped.
  - Concurrent action only one accepted if mutually exclusive.
- Chưa làm:
  - Không multi-server distributed consistency.

## Persistence Nâng Cao Sau Phase 8

### Mục tiêu

Thêm lưu trữ lâu dài khi foundation ổn định.

### Task P1: User persistence

- Loại: database.
- File/module dự kiến:
  - `Server.Infrastructure/Persistence/Users/*`
  - Migration sau khi được duyệt.
- Dependency: Auth decision.
- Tiêu chí hoàn thành:
  - Registered user lưu lâu dài.
  - Guest vẫn dùng được.
- Test cần thực hiện:
  - User repository integration test.
- Chưa làm:
  - Không social login nếu chưa cần.

### Task P2: Match result persistence

- Loại: database.
- File/module dự kiến:
  - `Server.Infrastructure/Persistence/Results/*`
  - `Server.Application/Results/RecordMatchResultHandler.cs`
- Dependency: Phase 4 result model.
- Tiêu chí hoàn thành:
  - Lưu game session completed.
  - Lưu player results.
- Test cần thực hiện:
  - Result persisted after game finished.
- Chưa làm:
  - Không rating/rank nếu chưa duyệt.

### Task P3: Action history persistence/replay-lite

- Loại: database.
- File/module dự kiến:
  - `Server.Infrastructure/Persistence/GameActions/*`
- Dependency: Stable action history model.
- Tiêu chí hoàn thành:
  - Lưu accepted action history.
  - Có thể reconstruct event timeline đơn giản.
- Test cần thực hiện:
  - Action history persisted in order.
- Chưa làm:
  - Không replay UI hoàn chỉnh.

## Checkpoints

### Checkpoint A: Sau Phase 1

- Card domain unit tests pass.
- Deck 52 lá không trùng.
- Shuffle/deal không mất bài.

### Checkpoint B: Sau Phase 2

- Room domain unit tests pass.
- Seat/ready/start invariants được bảo vệ.
- Room state machine rõ ràng.

### Checkpoint C: Sau Phase 4

- SimpleHighCard chạy qua unit tests.
- Game engine không phụ thuộc API/Unity.
- Private state filtering được test.

### Checkpoint D: Sau Phase 6

- Realtime integration tests pass.
- Command/event/snapshot hoạt động.
- Duplicate và invalid commands được xử lý.

### Checkpoint E: Sau Phase 8

- Hai client/test client chơi một ván hoàn chỉnh.
- Reconnect hoạt động.
- Privacy test pass.
- Foundation sẵn sàng để thêm game bài thật đầu tiên.

## Task Đầu Tiên Nên Implement Sau Khi Duyệt Spec

Task đầu tiên nên là Phase 0.2: chốt backend stack. Nếu chọn ASP.NET Core, task implement đầu tiên là tạo solution/backend project skeleton tối thiểu, sau đó Phase 1.1 `Card`, `Suit`, `Rank`.

Không nên bắt đầu bằng UI vì luật/state/auth ownership phải được định nghĩa trong domain và server trước.

## Quyết Định Cần Xác Nhận Trước Khi Code

- Backend stack: ASP.NET Core + SignalR có được chốt không?
- Unity client là frontend duy nhất trong giai đoạn đầu hay cần thêm web client?
- Game mẫu chọn `SimpleHighCard` hay variant "đánh hết bài trước thắng"?
- Guest-only trước hay cần registered login ngay từ đầu?
- Active room/session in-memory có được chấp nhận cho MVP không?
- Tie policy của SimpleHighCard: draw hay suit tiebreaker?
- Có cần lưu match result ngay trong phase đầu không, hay để sau Phase 8?
