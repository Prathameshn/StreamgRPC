This article will explain how to Use Streams in GRPC in a NodeJS Application.

To know the Basics of GRPC and Protocol Buffers you can read my Introduction to gRPC Article

What are Streams in GRPC
Streams in GRPC help us to send a Stream of messages in a single RPC Call.

In this article, we will be focussing on the following streams

Server Streaming GRPC: In this case, the client makes a single request to the server and the server sends a stream of messages back to the client.
Client Streaming GRPC: In this case, the client sends a stream of messages to the server. The server then processes the stream and sends a single response back to the client.
Server Streaming GRPC
Now let us create the server and client codes for a Server Streaming GRPC

Creating the .proto file
create a folder called proto. In the folder create a file called employee.proto. Copy the following into employee.proto

syntax = "proto3";

package employee;

service Employee {

  rpc paySalary (EmployeeRequest) returns (stream EmployeeResponse) {}
}


message EmployeeRequest {
  repeated int32 employeeIdList = 1;
}

message EmployeeResponse{
  string message = 1;
}
Refer to my grpc basics article to know more about .proto files and Protocol Buffers.

Here we are creating an rpc called paySalary which accepts EmployeeRequest as the request and sends stream of EmployeeResponse as the response. We use the keyword stream to indicate that the server will send a stream of messages

EmployeeRequest and EmployeeResponse are defined above as well. repeated keyword indicates that a list of data will be sent.

In this example, the request to paySalary will be a list of employee Ids. The server will respond with a stream of messages telling if Salary has been paid to an employee or not.

Creating Dummy data for the Server
Create a file called data.js and copy the following code into it.

//Hardcode some data for employees
let employees = [{
    id: 1,
    email: "abcd@abcd.com",
    firstName: "First1",
    lastName: "Last1"   
},
{
    id: 2,
    email: "xyz@xyz.com",
    firstName: "First2",
    lastName: "Last2"   
},
{
    id: 3,
    email: "temp@temp.com",
    firstName: "First3",
    lastName: "Last3"   
},
];

exports.employees = employees;
We will use it as the data source for the server.

Creating the Server
Create a file called as server.js. Copy the following code into server.js

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');


let packageDefinition = protoLoader.loadSync(
  PROTO_PATH,
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
let employee_proto = grpc.loadPackageDefinition(packageDefinition)
Refer to my grpc basics article to understand what the above script does.

Next, add the following piece of code into server.js

let { paySalary } = require('./pay_salary.js');

function main() {
  let server = new grpc.Server();
  server.addService(employee_proto.Employee.service, 
    { paySalary: paySalary }
  );
  server.bind('0.0.0.0:4500', grpc.ServerCredentials.createInsecure());
  server.start();
}

main();
In the above script, we are starting the GRPC Server and adding Employee Service into it along with paySalary implementation.

But paySalary function is defined in pay_salary.js file.

Let us create a pay_salary.js file.

Add the following script into pay_salary.js file

let { employees } = require('./data.js');
const _ = require('lodash');

function paySalary(call) {
    let employeeIdList = call.request.employeeIdList;
  
    _.each(employeeIdList, function (employeeId) {
      let employee = _.find(employees, { id: employeeId });
      if (employee != null) {
        let responseMessage = "Salary paid for ".concat(
          employee.firstName,
          ", ",
          employee.lastName);
        call.write({ message: responseMessage });
      }
      else{
        call.write({message: "Employee with Id " + employeeId + " not found in record"});
      }
  
    });
    call.end();
  
}

exports.paySalary = paySalary;
paySalary function takes call as the input. call.request will have the request sent by the client.

call.request.employeeIdList will have the list of employee Ids sent by the client.

We then loop over the EmployeeId’s and for each employee Id we do some processing.

For each employee Id, we call the call.write function at the last. call.write will write a single message in a stream back to the client.

In this case for each employee, call.write will send back whether salary has been paid or not.

Once all the Employee Id’s have been processed we call the call.end function. call.end indicates that the stream is complete.

Here is the final server.js file

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');


let packageDefinition = protoLoader.loadSync(
  PROTO_PATH,
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
let employee_proto = grpc.loadPackageDefinition(packageDefinition)

let { paySalary } = require('./pay_salary.js');

function main() {
  let server = new grpc.Server();
  server.addService(employee_proto.Employee.service, 
    { paySalary: paySalary }
  );
  server.bind('0.0.0.0:4500', grpc.ServerCredentials.createInsecure());
  server.start();
}

main();
Creating the Client
Create a file called client_grpc_server_stream.js. Copy the following code into the file.

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');

let packageDefinition = protoLoader.loadSync(
    PROTO_PATH,
    {keepCase: true,
     longs: String,
     enums: String,
     defaults: true,
     oneofs: true
    });
let employee_proto = grpc.loadPackageDefinition(packageDefinition).employee;
The above script has already been explained in my grpc basics article.

Next, add the following piece of the script to the client.

function main() {
  let client = new employee_proto.Employee('localhost:4500',
                                       grpc.credentials.createInsecure());
                                       
  let employeeIdList = [1,10,2];
  let call = client.paySalary({employeeIdList: employeeIdList});

  call.on('data',function(response){
    console.log(response.message);
  });

  call.on('end',function(){
    console.log('All Salaries have been paid');
  });

}

main();
client variable will have the stub which will help us call the function in the server.

employeeIdList is the input given to the server.

let call = client.paySalary({employeeIdList: employeeIdList}); script calls the paySalary function in the server and passes employeeIdList as the request. Since the server is going to send a stream of messages, call object will help us listen to the stream events

We listen to the ‘data’ event in call object for any message coming from the server in the stream. This is shown in the below script

  call.on('data',function(response){
    console.log(response.message);
  });
Here we just print the response message whenever we receive any message from the server

We listen to the ‘end’ event in the call object to know when the server stream ends. This is shown in the below script

  call.on('end',function(){
    console.log('All Salaries have been paid');
  });
Here when the stream ends we are printing ‘All Salaries have been paid’.

Here is the complete code for client_gprc_server_stream.js

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');

let packageDefinition = protoLoader.loadSync(
    PROTO_PATH,
    {keepCase: true,
     longs: String,
     enums: String,
     defaults: true,
     oneofs: true
    });
let employee_proto = grpc.loadPackageDefinition(packageDefinition).employee;

function main() {
  let client = new employee_proto.Employee('localhost:4500',
                                       grpc.credentials.createInsecure());
                                       
  let employeeIdList = [1,10,2];
  let call = client.paySalary({employeeIdList: employeeIdList});

  call.on('data',function(response){
    console.log(response.message);
  });

  call.on('end',function(){
    console.log('All Salaries have been paid');
  });

}

main();
Running the Code
Open a Command prompt and start the server using the following script.

node server.js
open a new command prompt and run the client using the following script.

node client_grpc_server_stream.js   
On running the client we will get the following output.

Salary paid for First1, Last1
Employee with Id 10 not found in record
Salary paid for First2, Last2
All Salaries have been paid
In this case, the client has sent 3 Id’s 1,10,2 to the server. The Server processes the Id’s one by one and sends a stream of messages to the client. Once all the messages in the stream are completed, the message ‘All Salaries have been paid’ is printed.

Client Streaming GRPC
Now let us create the server and client codes for a Client Streaming GRPC.

Creating the .proto file
In the previously created employee.proto file add the following

service Employee {

  rpc paySalary (EmployeeRequest) returns (stream EmployeeResponse) {}

  rpc generateReport (stream ReportEmployeeRequest) returns (ReportEmployeeResponse) {}
}

message ReportEmployeeRequest {
  int32 id = 1;
}

message ReportEmployeeResponse{
  string successfulReports = 1;
  string failedReports = 2;
}
Here we have added a new rpc called generateReport which accepts a stream of ReportEmployeeRequest as request and returns ReportEmployeeResponse as the response.

so the input to the rpc is a stream of employee Id’s and the response from the server will be a single response which says how many reports were generated and how many reports failed.

Here is the complete employee.proto file after our changes

syntax = "proto3";

package employee;

service Employee {

  rpc paySalary (EmployeeRequest) returns (stream EmployeeResponse) {}

  rpc generateReport (stream ReportEmployeeRequest) returns (ReportEmployeeResponse) {}
}


message EmployeeRequest {
  repeated int32 employeeIdList = 1;
}

message EmployeeResponse{
  string message = 1;
}

message ReportEmployeeRequest {
  int32 id = 1;
}

message ReportEmployeeResponse{
  string successfulReports = 1;
  string failedReports = 2;
}
Creating the Server
Here is the complete server.js code with the new rpc added

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');


let packageDefinition = protoLoader.loadSync(
  PROTO_PATH,
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
let employee_proto = grpc.loadPackageDefinition(packageDefinition).employee;


let { paySalary } = require('./pay_salary.js');
let { generateReport } = require('./generate_report.js');

function main() {
  let server = new grpc.Server();
  server.addService(employee_proto.Employee.service, 
    { paySalary: paySalary ,
      generateReport: generateReport }
  );
  server.bind('0.0.0.0:4500', grpc.ServerCredentials.createInsecure());
  server.start();
}

main();
In the above script, we can see that we have added generateReport function as well to the grpc server. Also we can see that generateReport function comes from generate_report.js file.

Create a file called as generate_report.js.

Add the following script into the file

let { employees } = require('./data.js');
const _ = require('lodash');

function generateReport(call, callback){

    let successfulReports = [];
    let failedReports = [];
    call.on('data',function(employeeStream){
        let employeeId = employeeStream.id;
        let employee = _.find(employees, { id: employeeId });
        if (employee != null) {
          successfulReports.push(employee.firstName);
        }
      else{
          failedReports.push(employeeId);
      }

    });
    call.on('end',function(){
        callback(null,{
            successfulReports: successfulReports.join(),
            failedReports: failedReports.join()
        })
    })
}

exports.generateReport = generateReport;
The generateReport function takes two inputs, call and callback

In order to get the stream of messages from the client, we need to listen to the data event in the call object. This is done in the following script.

    call.on('data',function(employeeStream){
        let employeeId = employeeStream.id;
        let employee = _.find(employees, { id: employeeId });
        if (employee != null) {
          successfulReports.push(employee.firstName);
        }
      else{
          failedReports.push(employeeId);
      }

    });
The data event is called for every single message coming from the client. The message is present in the employeeStream variable. On receiving the message we try to generate a report and find out if it succeeded or failed.

The end event on the call object indicates that the client stream has ended. The following code shows how to listen to the end event.

 call.on('end',function(){
        callback(null,{
            successfulReports: successfulReports.join(),
            failedReports: failedReports.join()
        })
    })
In this case, when the end event happens, we combine all the success and failure reports into a single response object and send it back to the client using the callback object.

Creating the Client
Create a file called as client_grpc_client_stream.js. Add the following script into it.

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');
const _ = require('lodash');

let packageDefinition = protoLoader.loadSync(
  PROTO_PATH,
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
let employee_proto = grpc.loadPackageDefinition(packageDefinition).employee;
The above script does the same functionality as we saw in the server code.

Add the following script as well to the client_grpc_client_stream.js.

function main() {
  let client = new employee_proto.Employee('localhost:4500',
    grpc.credentials.createInsecure());

  let call = client.generateReport(function (error, response) {
    console.log("Reports successfully generated for: ", response.successfulReports);
    console.log("Reports failed since Following Employee Id's do not exist: ", response.failedReports);
  });

  let employeeIdList = [1, 10, 2];
  _.each(employeeIdList, function (employeeId) {
        call.write({ id: employeeId });
  })

  call.end();
}

main();
Let us see what the script above is doing.

let call = client.generateReport(function (error, response) {
    console.log("Reports successfully generated for: ", response.successfulReports);
    console.log("Reports failed since Following Employee Id's do not exist: ", response.failedReports);
  });
In this portion of the script, we are creating a call object and calling the generateReport function. Also inside the generateReport function, we are indicating what the client should do once it receives the response from the server. In this case, we are printing the successful and failed reports which the server sends back.

 let employeeIdList = [1, 10, 2];
  _.each(employeeIdList, function (employeeId) {
        call.write({ id: employeeId });
  })
In the above portion of the script, we are looping over the employee IDs and sending a stream of messages to the server. We use call.write to send the message in a stream to the server.

Finally, once we have sent all the messages in a stream, we indicate that the stream is complete using the call.end function as shown below

call.end();
The complete code for client_grpc_client_stream is given below.

const PROTO_PATH = __dirname + '/proto/employee.proto';

const grpc = require('grpc');
const protoLoader = require('@grpc/proto-loader');
const _ = require('lodash');

let packageDefinition = protoLoader.loadSync(
  PROTO_PATH,
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  });
let employee_proto = grpc.loadPackageDefinition(packageDefinition).employee;

function main() {
  let client = new employee_proto.Employee('localhost:4500',
    grpc.credentials.createInsecure());

  let call = client.generateReport(function (error, response) {
    console.log("Reports successfully generated for: ", response.successfulReports);
    console.log("Reports failed since Following Employee Id's do not exist: ", response.failedReports);
  });

  let employeeIdList = [1, 10, 2];
  _.each(employeeIdList, function (employeeId) {
        call.write({ id: employeeId });
  })

  call.end();
}

main();
Running the Code
Open a Command prompt and start the server using the following script.

node server.js
open a new command prompt and run the client using the following script.

node client_grpc_client_stream.js   
On running the client we will get the following output.

Reports successfully generated for:  First1,First2
Reports failed since Following Employee Id\'s do not exist:  10
In this case, the client has sent 3 Id’s 1,10,2 to the server as a stream of messages. The server then processes the messages in the stream and sends a single response back to the client showing how many reports succeeded and how many reports failed.

Code
The Code discussed in this article can be found here

References
GRPC official Documentation : https://grpc.io/

Protocol Buffers Proto3 documentation : https://developers.google.com/protocol-buffers/docs/proto3

