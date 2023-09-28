```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, meta = (GetOptions = "GetNameOptions"))
FName Name;

UFUNCTION(CallInEditor)
TArray<FString> GetNameOptions() const
{
    return { TEXT("N1"), TEXT("N2"), TEXT("N3") };
}
```