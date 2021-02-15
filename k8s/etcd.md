## etcd ( & raft )
여러 개의 서버를 하나의 서버처럼 사용하게 할 수 있게 하기 위해서는 여러 서버에서 클라이언트에서 입력한 데이터 값에 대한 합의가 필요함 -> raft algorithm

### Quorum
- 작업을 끝냈다고 간주하는 cnt 
- (서버 개수 / 2 + 1) 

### State
- Leader : clinet로 부터 요청을 받는 서버
- Follower 
- Candidate : leader 후보

### Timer
- heartbeat timeout : (모든 서버는 leader로 heartbeat를 일정 주기로 받는다)
- election timeout : follower 가 candidate 가 되기까지 걸리는 시간 (heartbeat timeout 발생 후) 

