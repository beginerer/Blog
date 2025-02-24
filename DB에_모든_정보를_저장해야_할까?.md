# db에 모든 정보를 저장해야 할까?
텍사스 홀덤 서비스 구현에서 게임 방 상태에대한 모든 정보를 db에 저장해야 되는지 의문이 들었다. 

room entity => roomState dto로 변환하는 매핑 메서드를 수정하는 과정에서 room 엔티티가 roomState에 필요한 모든 정보를 제공할 수 없다는 사실을 뒤늦게 알게되었다. 
이 문제를 해결하기 위해서 방에 대한 상태정보를 모두 db에 저장해야되는지 고민을했다. 

db에 게임 방 정보를 모두 저장하면 entity -> dto 매핑과정이 편리하고, 데이터의 일관성을 보장할 수 있다는 장점이 있다. 하지만 잦은 db 접근은 실시간 처리에 악영향을 주며, 서버 비용적인 측면에서도 비효율적이다.

뿐만아니라 비지니스적인 도메인 영역과 UI 상태정보 영역이 함께 저장되는 것은 역할과 책임의 관점에서 객체지향적이지 못하고, 관심사의 분리가 제대로 이루어지지 않아 불필요한 데이터가 영속될 수 있는 문제가 발생한다.

그래서 ui상태정보 같은 덜 중요한 정보들은 레디스에 저장하기로 결정했다. 메모리 방식이기때문에 데이터의 영속성이 깨질 수 있는 단점이 분명 존재하지만 비지니스와 관련된 중요한 도메인 정보는 db에 저장되기때문에 문제 없다고 판단했다. 

더구나 나중에 게임 이용자가 많아져서 서버를 스케일아웃 하는 상황에서도 유연하게 대응할 수 있다고 생각했다.

따라서, 처음 방을 생성하고(DB를 통한) dto객체를 반환할때와 이미 방이 생성된 상태에서 dto를 반환하는 경우를 구분하여 메서드를 만들었다. 
그리고 방 삭제 api가 호출되면 트랜잭션 안에서 레디스 정보도 동시에 삭제되게 만들어서 같은 생명주기를 갖도록 구현했다.

마지막으로 DB와 Redis는 서로 다른 트랜잭션 관리 메커니즘을 사용해서 하나의 트랜잭션으로 관리되기 어렵기 때문에 데이터의 원자성이나 일관성 문제가 발생할 여지가 있다. 
이부분은 아직 부족하여 추후에 공부후에 업데이트할 예정이다.

```java
public RoomState createNewRoomState(Room room) {
	RoomState roomState = new RoomState();
	roomState.setRoomId(room.getRoomId);
	...
	return roomState;
}
```

```java
public RoomState toRoomState(Room room) {
	// 레디스에 저장된 RoomState 정보를 조회하여 반환
	RoomState roomState = roomStateService.find(roomId,RoomState.class);
	return roomState
}
