```cpp
UCLASS(Blueprintable)
class QUESTTOOL_API UXBlueprint : public UObject, public IConfigSerializable
{
	GENERATED_BODY()

public:
	UPROPERTY()
	UXGraph* Graph = nullptr;
};
```