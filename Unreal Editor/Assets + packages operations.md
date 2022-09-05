#### Get assets by class
```cpp
#include "AssetRegistry/AssetRegistryModule.h"

const FAssetRegistryModule& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");

TArray<FAssetData> AssetsData;
AssetRegistryModule.GetAssetsByClass(UAssetClass::StaticClass()->GetFName(), AssetsData, bSearchSubClasses);

for (const FAssetData& AssetData : AssetsData)
{
	UAssetClass* Asset = Cast<UAssetClass>(AssetData.GetAsset());
}
```

#### Get assets by package
```cpp
const FAssetRegistryModule& AssetRegistryModule =  FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");
IAssetRegistry& AssetRegistry = AssetRegistryModule.Get();

TArray<FAssetData> AssetsData;
AssetRegistry.GetAssetsByPackageName(Package->GetFName(), AssetsData, bIncludeOnlyOnDiskAssets);
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

#### Rename asset
```cpp
TSet<UPackage*> ObjectsUserRefusedToFullyLoad;
FText ErrorMessage;
ObjectTools::FPackageGroupName PGN;
PGN.ObjectName = TEXT("smthg");
PGN.GroupName = TEXT("");
PGN.PackageName = TEXT("smthg");
ObjectTools::RenameSingleObject(Package, PGN, ObjectsUserRefusedToFullyLoad, ErrorMessage);
```

#### Save asset
```cpp
const FString PackageName = FPackageName::LongPackageNameToFilename(Package->GetPathName(), FPackageName::GetAssetPackageExtension());
UPackage::SavePackage(Package, Asset, EObjectFlags::RF_Public | EObjectFlags::RF_Standalone, *PackageName);
```

#### Save package
```cpp
TArray<UPackage*> PackagesToSave;
FEditorFileUtils::PromptForCheckoutAndSave(PackagesToSave, bCheckDirty, bPromptToSave);
```

#### Remove asset from package
```cpp
Asset->SetFlags(RF_Transient);
// And save package
```
