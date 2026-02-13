## CMC 主要类分析


### UCharacterMovementComponent

- 继承自 `UPawnMovementComponent << UNavMovementComponent << UMovementComponent`


##### 主要属性:

`FNetworkPredictionData_Client_Character* ClientPredictionData`
`FNetworkPredictionData_Server_Character* ServerPredictionData`
- todo...

`FRootMotionSourceGroup CurrentRootMotion`


##### 主要方法:

---

### FNetworkPredictionData_Client_Character

##### 主要属性:

`float ClientUpdateTime`

`float CurrentTimeStamp`


`float LastReceivedAckRealTime`

`TArray<FSavedMovePtr> SavedMoves`	
`TArray<FSavedMovePtr> FreeMoves`	
`FSavedMovePtr PendingMove`			
`FSavedMovePtr LastAckedMove`		


`int32 MaxFreeMoveCount`
`int32 MaxSavedMoveCount`

---

### FSavedMove_Character


--- 

### Root Motion 

区分 HasAnimRootMotion
和 CurrentRootMotion.HasActiveRootMotionSources()