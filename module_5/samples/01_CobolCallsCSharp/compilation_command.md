dotnet publish -c Release -r linux-x64

cd /home/trainees/codes/cobol-training/module_5/samples/01_CobolCallsCSharp

## ${your_dir}/01_CobolCallsCSharp

cobc -x -free -fstatic-call CobolClient.cob -LCSharpLib/bin/Release/net8.0/linux-x64/publish -l:CSharpLib.so -o CobolClient


LD_LIBRARY_PATH=CSharpLib/bin/Release/net8.0/linux-x64/publish:$LD_LIBRARY_PATH ./CobolClient

