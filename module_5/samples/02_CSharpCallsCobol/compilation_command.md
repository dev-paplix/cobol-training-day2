cd /home/trainees/codes/cobol-training/module_5/samples/02_CSharpCallsCobol

## ${your_dir}/02_CSharpCallsCobol

cobc -m -free CobolEngine.cob -o libCobolEngine.so

cd DotNetClient

LD_LIBRARY_PATH=..:$LD_LIBRARY_PATH dotnet run

