#### Get assets by class
```cpp
#include "AssetRegistry/AssetRegistryModule.h"

const FAssetRegistryModule& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");

TArray<FAssetData> AssetsData;
AssetRegistry.GetAssetsByClass(UAssetClass::StaticClass()->GetFName(), AssetsData, bSearchSubClasses);

for (FAssetData& AssetData : AssetsData)
{
	UAssetClass* Asset = Cast<UAssetClass>(AssetData.GetAsset());
}
```

#### Load package
```cpp
const FString PackageName = Object->GetPathName().Replace( 
    *FString::Printf(TEXT(".%s"), *Object->GetName()), TEXT(""))
UPackage* Package = FindPackage(nullptr, *PackageName);
```

#### Create package
```cpp
const FString PackageName = Object->GetPathName().Replace( 
    *FString::Printf(TEXT(".%s"), *Object->GetName()), TEXT(""))
CreatePackage(*PackageName);
```

#### Change asset package
```cpp
Asset->Rename(*Asset->GetName(), NewPackage);
```

#### Load asset
```cpp
UAssetClass* Asset = LoadObject<UAssetClass>(Package, *AssetPath)
```

#### Create asset
```cpp
UAssetClass* Asset = NewObject<UAssetClass>(Package, *AssetName, RF_Public | RF_Standalone);
```

#### Save asset
```cpp
const FString SoundPackageName = FPackageName::LongPackageNameToFilename(Package->GetPathName(), FPackageName::GetAssetPackageExtension());
UPackage::SavePackage(Package, Asset, EObjectFlags::RF_Public | EObjectFlags::RF_Standalone, *PackageName);
```

#### Remove asset from package
```cpp
Asset->SetFlags(RF_Transient);
// And save package
```
