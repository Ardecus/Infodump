#### Property to be customized

```cpp
USTRUCT()
struct FPropertyName 
{
	GENERATED_BODY()
	...
};
```

#### Actual customization code

PropertyNameCustomization.h
```cpp
#include "IPropertyTypeCustomization.h"

class FPropertyNameCustomization : public IPropertyTypeCustomization
{
public:
	static TSharedRef<IPropertyTypeCustomization> MakeInstance();

	void CustomizeHeader(TSharedRef<IPropertyHandle> StructPropertyHandle, FDetailWidgetRow& HeaderRow, IPropertyTypeCustomizationUtils& StructCustomizationUtils) override;
	void CustomizeChildren(TSharedRef<IPropertyHandle> StructPropertyHandle, IDetailChildrenBuilder& StructBuilder, IPropertyTypeCustomizationUtils& StructCustomizationUtils) override;
}
```

PropertyNameCustomization.cpp
```cpp

TSharedRef<IPropertyTypeCustomization> FPropertyNameCustomization::MakeInstance()
{
	return MakeShared<FPropertyNameCustomization>();
}

// To change customization of whole property
void FPropertyNameCustomization::CustomizeHeader(TSharedRef<IPropertyHandle> StructPropertyHandle, FDetailWidgetRow& HeaderRow, IPropertyTypeCustomizationUtils& StructCustomizationUtils)
{
	// Leaves it as-is
	LastChangedPropertyHandle = &StructPropertyHandle.Get();
	HeaderRow.NameContent()
	[
		StructPropertyHandle->CreatePropertyNameWidget()
	];
}

// To change customization of property subproperties
void FPropertyNameCustomization::CustomizeChildren(TSharedRef<IPropertyHandle> StructPropertyHandle, IDetailChildrenBuilder& StructBuilder, IPropertyTypeCustomizationUtils& StructCustomizationUtils)
{
	// Slate goes here
	...
	// For custom sub-property view
	const TSharedPtr<IPropertyUtilities> PropertyUtils = StructCustomizationUtils.GetPropertyUtilities();
	uint32 NumberOfChild;
	if (StructPropertyHandle->GetNumChildren(NumberOfChild) == FPropertyAccess::Success)
	{
		for (uint64 FieldIndex = 0; FieldIndex < NumberOfChild; ++FieldIndex)
		{
			const TSharedRef<IPropertyHandle> ChildPropertyHandle = StructPropertyHandle->GetChildHandle(FieldIndex).ToSharedRef();
			StructBuilder.AddProperty(ChildPropertyHandle)
			// Slate goes here
			.ShowPropertyButtons(bShowPropertyButtons)
			.IsEnabled(MakeAttributeLambda([=]
			{
				return !StructPropertyHandle->IsEditConst() && PropertyUtils->IsPropertyEditingEnabled();
			}));
			;
		}
	}
	...
	// Get selected properties
	TArray<void*> PropertyValues;
	StructPropertyHandle->AccessRawData(PropertyValues);
	FPropertyName* FirstSelectedEditedProperty{ static_cast<FPropertyName*>(PropertyValues[0]) };
	
}
```