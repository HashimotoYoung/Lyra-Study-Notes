
## 1.分析Struct的同步机制
```cpp
USTRUCT()
struct FMyStruct
{
    GENERATED_BODY()
    
    UPROPERTY(Replicated)
    int32 Health;
    
    UPROPERTY(Replicated)
    int32 mana;
    
    UPROPERTY()
    float Speed;
    
    FMyStruct()
    {
        Health = 100;
        mana = 50;
        Speed = 10.0f;
    }
};

UCLASS()
class MYGAME_API ASimpleActor : public AActor
{
    GENERATED_BODY()
    
public:    
    ASimpleActor();

protected:
    UPROPERTY(Replicated)
    FMyStruct PlayerData;

    virtual void BeginPlay() override;

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    
    // Test function
    UFUNCTION(CallInEditor = true)
    void TestChanges();
};

// SimpleActor.cpp
#include "SimpleActor.h"

ASimpleActor::ASimpleActor()
{
    PrimaryActorTick.bCanEverTick = false;
    bReplicates = true;
    
    // Initialize struct
    PlayerData.Health = 100;
    PlayerData.mana = 50;
    PlayerData.Speed = 10.0f;
}

void ASimpleActor::BeginPlay()
{
    Super::BeginPlay();
    
    UE_LOG(LogTemp, Warning, TEXT("Initial - Health: %d, Mana: %d, Speed: %f"), 
           PlayerData.Health, PlayerData.mana, PlayerData.Speed);
}

void ASimpleActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    DOREPLIFETIME(ASimpleActor, PlayerData);
}

void ASimpleActor::TestChanges()
{
    if (HasAuthority())
    {
        // Server changes - will replicate to clients
        PlayerData.Health = 75;
        PlayerData.mana = 25;
        
        // Local change - won't replicate
        PlayerData.Speed = 20.0f;
        
        UE_LOG(LogTemp, Warning, TEXT("Server changed - Health: %d, Mana: %d, Speed: %f"), 
               PlayerData.Health, PlayerData.mana, PlayerData.Speed);
    }
}

```
#### struct with NetSerialzie
```cs
USTRUCT()
struct FMyStruct
{
    GENERATED_BODY()
    
    UPROPERTY(Replicated)
    int32 Health;
    
    UPROPERTY(Replicated)
    int32 mana;
    
    UPROPERTY()
    float Speed;

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms);
};

bool FMyStruct::NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
{
    // All properties in one atomic update
}
```

---
## 2.分析Transform相关
```cpp
void ACustomPlayerController::Move(const FInputActionValue& inputValue)
{
	const FVector2D inputAxisVector = inputValue.Get<FVector2D>();
	const FRotator rotation = GetControlRotation();
	const FRotator YawRotation(0, rotation.Yaw, 0);
    //rotation diff with FRotationMatrix(rotaion)
	const FVector ForwardDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
	const FVector rightDirection = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);

	if (APawn* ap = GetPawn<APawn>()) {
		ap->AddMovementInput(ForwardDirection, inputAxisVector.Y);
		ap->AddMovementInput(rightDirection, inputAxisVector.X);
	}
}
```