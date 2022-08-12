## Environment Requirements
* Linux operating system
* Java 11
* Dotnet core 3.1
## Target game Requirements
* Unity-based game
* Compiled with IL2CPP mode
* Use Unity IAP library for implementing in-game purchase
## PaymentScope File Structure
```bash
PaymentScope
├── src (1)
│   ├── ghidra_scripts (1A)
│   ├── Il2CppDumper-Customized (1B)
│   └── python (1C)
└── Tools (2)
    ├── Il2CppDumper-Customized (2A)
    └── ghidra_10.0.3_PUBLIC (2B)
```
This is the file structure of PaymentScope. At a high level, there are two parts (1) the PaymentScope code; (2) the external tools that PaymentScope depends on. 
* (1) PaymentScope code
	* (1A) It is the core implementation of PaymentScope which is in Ghidra Script format. It cannot be directly executed and relies on Ghidra to run. The class `TaintAnalysisPayment` is the starting point of the Ghidra Script. Please note that this folder needs to be moved to `~/ghidra_scripts` for running which is required by Ghidra.
	* (1B) The source code of customized Il2CppDumper. In particular, we customized it to extract more information such as all the classes and their fields from the games.
	* (1C) The wrapped PaymentScope which calls Ghidra to analyze provided APK or on the existing cached output file. **It is the main entrance of PaymenScope**.
* (2) The external tools. Please note that the folder names in `Tools` cannot be modified because (1C) relies on them.
	* (2A) The customized Il2CppDumper. It will extract the symbol information for game files
	* (2B) Ghidra framework. It analyzes the given binary and applies (1A) to it. Please note that PaymenScope needs json.jar to run which is missing from Ghidra, we have added it which is located at `ghidra_10.0.3_PUBLIC/Ghidra/patch/json-20200518.jar`. In the recent tests, we use this particular version. **To use another version of Ghidra, you need to add this jar file and modify the path of Ghidra in (1C).**

## How PaymentScope works
PaymentScope depends on Ghidra for analysis. There are three independent steps when analyzing one game binary.
* (S1) Extracting the symbols from the game file by using Il2CppDumper. **Time consumption: less than a half minute.**
* (S2) Ghidra preprocesses the game binary with symbols and generates a Ghidra project. **Time consumption: 20 minutes to several hours (very slow).**
* (S3) PaymentScope works as Ghidra Script to analyze the game binary to detect vulnerability with Ghidra. **Time consumption: less than one minute.**
Please note that the estimated time consumption is based on Ghidra 10.0.3. It may change for other versions.

## How to run PaymenScope
### Configuration
* Move  (1A) to folder `~/ghidra_scripts`
* Set the absolute path of (2) in file `src/python/config.json`. For example: 
```json
{
    "tools_folder" : "/home/paymentscope/Tools"
}
```
### Command line parameters
```bash
$ ./src/python/paymentScope.py --help
usage: paymentScope.py [-h] [-n NEW_PROCESS_MODE] [-c CACHED_MODE] [-o OUTPUT]
                       [-p PACKAGE_NAME]

optional arguments:
  -h, --help            show this help message and exit
  -n NEW_PROCESS_MODE, --new_process_mode NEW_PROCESS_MODE
                        The path of the target APK
  -c CACHED_MODE, --cached_mode CACHED_MODE
                        Run the script on an existing output folder (cached)
  -o OUTPUT, --output OUTPUT
                        The output folder where the project will be stored
  -p PACKAGE_NAME, --package_name PACKAGE_NAME
                        The package name of the target game
```
As mentioned, there are three steps when analyzing one game binary. While the S2 can take several hours, PaymentScope analysis (S3) only needs one minute. Therefore, the PaymentScope python wrapper supports two modes.
#### (M1) New processing mode which runs S1, S2, and S3 on a given game file and generates an output project. (very slow)
There are three required parameters for this mode:
* `-n` which specifies the path of the target game APK.
* `-o` which specifies the output folder where the generated project will be stored.
* `-p` which specifies the package name of the target game
example:
```bash
./src/python/paymentScope.py \
    -n /home/paymentscope/Desktop/TestData/apks/com.test.game.apk \
    -o /home/paymentscope/Desktop/TestData/outputs/com.test.game \
    -p com.test.game
```
#### (M2) Cached mode which runs PaymentScope analysis (S3) on an existing output project. 
There are two required parameters for this mode:
* `-c` which specifies the project folder that generated by **new processing mode**
* `-p` which specifies the package name of the target game
example:
```bash
./src/python/paymentScope.py \
    -c /home/paymentscope/Desktop/TestData/outputs/com.test.game \
    -p com.test.game
```
### Output
The output of PaymentScope is a folder (i.e., project folder). The following is the file structure of one such folder and the explanation of the files:
```bash
.
├── com.test.game_global-metadata.dat    # The encoded symbol file from game APK
├── com.test.game_libil2cpp.so	          # The game binary from game APK
├── classes.json                                  # Il2CppDumper generated file
├── DummyDll                                      # Il2CppDumper generated files
├── dump.cs                                       # Il2CppDumper generated file
├── il2cpp.h                                      # Il2CppDumper generated file
├── script.json                                   # Il2CppDumper generated file
├── stringliteral.json                            # Il2CppDumper generated file    
├── ghidra                                        # Ghidra generated files
└── analysisRes.json                              # The final analysis result file    
```
Among them, the final analysis result file is the most important one. Here  is an example file:
```json
{
    "nodes": {
        "0": {
            "depth": 0,
            "children": [1],
            "contextMethod": "UnityEngine.Purchasing.Product$$get_receipt",
            "id": 0,
            "unityBuildin": false
        },
...
        "3": {
            "depth": 3,
            "currentPcode": " ---  CALL (ram, 0xf1a5a8, 8) , (register, 0x4000, 8) , (const, 0x0, 8)",
            "leafType": "Ending_API",
            "contextMethod": "PurchaseController$$ProcessPurchase",
            "pcodeOffset": "009f5538",
            "additionalInfo": "UnityEngine.Debug$$Log",
            "id": 3,
            "leafNote": "UnityEngine.Debug$$Log",
            "parentID": 2,
            "unityBuildin": false
        }
    },
    "executablePath": "/home/cszuo/..._libil2cpp.so",
    "leafes": [3],
    "pname": "com.kbpro.MashaPuzzles",
    "leafInfo": {"Ending_API": ["UnityEngine.Debug$$Log [3]"]},
    "isVulnerable": "no-verification"
}
``` 
There are several fields in it and the following ones are important:
* `nodes` contains all the nodes of the tained paths. The meaning of its subfields is indicated by their names.
* `leafInfo` contains the node types of leaf nodes which are used to decide the payment verification type. 
* `isVulnerable` it indicates the payment verification type. `no-verification`, `local-verification`, or `remote-verification`. For ethical reason, we have set this field to `None`.

