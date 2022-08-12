#### NewAsset.h
```cpp
UCLASS()
class UNewAsset : public UObject
{
	GENERATED_BODY()
};
```

#### NewAssetTypeActions.h
```cpp
#include "AssetTools/Public/AssetTypeActions_Base.h"

class FNewAssetTypeActions : public FAssetTypeActions_Base
{
public:
	FNewAssetTypeActions(EAssetTypeCategories::Type InAssetCategory) :
		SupportedClass(UNewAsset::StaticClass())
		NewAssetTypeCategory(InAssetCategory)
	{}

	FText GetName() const override { return Name; }
	FColor GetTypeColor() const override { return TypeColor; }
	UClass* GetSupportedClass() const override { return SupportedClass; }
	uint32 GetCategories() override { return NewAssetTypeCategory; }

private:
	FText Name = Const::SomeName;
	FColor TypeColor = FColor::Orange;
	UClass* SupportedClass = nullptr;
	EAssetTypeCategories::Type NewAssetTypeCategory;
};
```

#### NewAssetFactory.h
```cpp
#include "Factories/Factory.h"

UCLASS()
class UNewAssetFactory : public UFactory
{
	GENERATED_BODY()

public:
	UNewAssetFactory() :
		SupportedClass(UNewAsset::StaticClass())
	{}

	virtual UObject* FactoryCreateNew(UClass* InClass, UObject* InParent, FName InName, EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn) override
	{
		return NewObject<UNewAsset>(InParent, InClass, InName, Flags | RF_Transactional);
	}
};
```

#### Style definition for icons
```cpp
StyleSet->Set("ClassIcon.NewAsset", new IMAGE_BRUSH("Icon_16", Icon16x16));
StyleSet->Set("ClassThumbnail.NewAsset", new IMAGE_BRUSH("Icon_64", Icon64x64));
```
More on style l8r