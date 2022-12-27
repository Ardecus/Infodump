#### Declare
```cpp
TAutoConsoleVariable<ValueType> VariableNameInCode(TEXT("VariableNameInConsole"),
	DefaultValue,
	TEXT("VariableDescription"),
	EConsoleVariableFlags /*variable type, ECVF_Default as default*/);
```
#### Use
```cpp
VariableNameInCode->GetValue(OutParamValue)
// or better
VariableNameInCode->GetBool/Int/Float/String()
```