cd /home/trainees/codes/cobol-training/module_5/samples/03_CaseStudy_AspNetCore/CobolBackend

## ${your_dir}/03_CaseStudy_AspNetCore/CobolBackend

cobc -m -free LoanRiskEngine.cob -o libLoanRiskEngine.so

cd ../WebApi

## Run built-in test suite:
LD_LIBRARY_PATH=../CobolBackend:$LD_LIBRARY_PATH dotnet run -- --test

## Run web API server:
LD_LIBRARY_PATH=../CobolBackend:$LD_LIBRARY_PATH dotnet run

