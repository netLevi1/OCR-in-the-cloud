# OCR-in-the-cloud
### Using AWS (EC2, SQS, S3) to run OCR algorithm distributively

#### Simplefied Illustration:
![image](https://github.com/netLevi1/OCR-in-the-cloud/assets/90900617/ca28f93e-74df-4424-85e2-700d1c07a852)
1. 'Local application' uploads the input file to S3.
2. 'Local application' starts the 'Manager' EC2 instance.
3. 'Manager' downloads the input file from S3.
4. 'Manager' starts EC2 'Workers' instances that perform the algorithm.
5. After completion, 'Workers' send the output to the 'Manager'.
6. 'Manager' combines all 'Workers' results into a single file and uploads it to S3.
7. 'Manager' updates 'Local application' that the task has been completed.
8. 'Local application' downloads the output file from S3.

#### How to run:

First, you need to create the jar file with dependencies.(the manager and worker jar files are already uploaded to S3)
It can be done easly using intellij. open the project in intellij, and click on the 'Maven' bar to the left. Then choose 'lifecycle' and click on: 'package'.
It will create jar file with dependencies which yuo neet to run using the following command:
*Java -jar jarFileName.jar inputfile.txt outputfile.txt n terminate*
Where 'n' is the  number of URLs per Worker and 'terminate' is optional.
Terminate is either "terminate" or every other string – where terminate means finish all tasks and terminate the project and every other string means do the work and wait for other Local Applications.  
In this Project we use ami-0b0dcb5067f052a63 with the type T2-MICRO .

#### How the program works:

We run the command:

*Java -jar jarFileName.jar inputfile.txt outputfile.txt 5 terminate*

Then the flow of the program is:

##### Local Application:
- initialize the services (**EC2** client, **S3** client, **SQS** client).
- create a new **EC2** 'Manager' instance.
- create 1 **SQS** Queue: 'Local_to_Manager' which send messages from Local Application to Manager.
- Upload the input file to **S3**. (in our case, the input file contains a list of links)
- Send a New Task message to the Manger using the Local_to_Manager **SQS** Queue.

##### Manager:
- initialize the services (EC2 client, S3 client, SQS client).
- initialize WorkerCounterThread which makes sure that when worker goes down unintentionally, another one will take its place.
- check for new Task messages on **SQS** and for each one it will open a Manager Thread.

##### Manager Thread:
 - initialize **SQS** queues:
<br>'managerToLocal'+'id' queue which is the queue that related to that specific localApplication.
<br>'manager_workers' 
<br>'workers_manager
- starts Worker_to_Manger_Thread which is the thread that check for doneOCR messages from the workers.

- download the input file from **S3**
- send the URLs as messages to the Manager_to_Worker Queue.
- Create workers.
- check if terminate message recieved from LocalApplication.
- if terminate message recived, it waits until all workers done their jobs and terminates them, upload the output file to S3 and send ProcessIsDone to the LocalApplication.

##### Workers:
- Receive the message with the URL. 
- Perform the OCR algorithm.
- Return the output to the Worker_to_Manager Queue with the message "done OCR".
##### Worker_to_Manager_Thread:
- Receive the "done OCR" message and write the output to output.txt file.
##### Locl_to_Manager_Thread:
- Wait until all the orkers is done and send a message to the local "process is done".
- And if terminate message was received it terminates the workers/

##### Local Application:
- the localApplication receives "process is done " messages and download the output file from **S3** and create the html file and save it to local memory.

In this example the program finished in about 4.5 minutes.

#### Additional information: 

- credentials are stored locally on the pc where the localApp is run from. There is no need to send credentials in plain text to the instances.
- the program is scalable. For each LocalApp that sends a newTask message, we open a new thread that handles with it.
- we have a special thread that all it does is to check if the number of running workers matches the number of workers that has been opened, and open a new worker if needed.
- threads in our application is essentially a good idea because it lets the manager handle several task simultaneously. it also depends on which system the manger is running on, and how many local applications are expected to be. If the manger instance runs on a low spec computer and it shuold handle large amount of clients, performance will suffer.
- when the workers finished and terminate is accepted, the manger terminates all workers and then terminate inself.
  
