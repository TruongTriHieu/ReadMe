# Message-queue
- A standard interface to access different message queues. Both the publisher and the subscriber are event emitters. Message Queue is a Queue, containing many Messages.

## Prerequisites
### Create new private message
``` js
1. Open Computer Management
2. Open Private Queues -> New Private Queues
3. To Queue Name
4. Check the Transactional box.
```
### Setting
- Add each key and value for each queue registered in the computer management.The MSMQ queue name is specified in an appSettings section of the configuration file, as shown in the following sample configuration.

Say what the step will be:
``` js
Give examples: 
< appSettings > 
    < add  key = " orderQueueName "  value = " . \ private $ \ Order " /> 
</ appSettings >
```
And repeat: 
``` js
  add key="RUNNING_MESSAGE_QUEUE_NAME" value=".\private$\running_process_queue"
  add key="CLOSED_MESSAGE_QUEUE_NAME" value=".\private$\closed_process_queue"
  add key="KILL_MESSAGE_QUEUE_NAME" value=".\private$\kill_process_queue"
```
> [!NOTE]
> The queue name uses a dot (.) for the local computer and backslash separators in its path.

### Installing
- The service receives messages from the queue and processes orders. The service creates a transactional queue and sets up a message received message handler, as shown in the following sample code.
``` js
   _queueName = ConfigurationManager.AppSettings.Get("RESTART_MESSAGE_QUEUE_NAME");
```
- When a message is received in the queue, the ProcessMessage message processor is called and the terminal is checked.
``` js
  //Process Message
        public void ProcessMessage(string message)
        {
            try
            {
                var data = JsonConvert.DeserializeObject<Account>(message);
                if (data.Version == MetaTrade.Domain.Enums.TerminalVersion.MT5)
                {
                    _tiktikTerminal = new MT5Terminal();
                }
                else
                {
                    // Instance terminal version 4
                    _tiktikTerminal = new MT4Terminal();
                }


                if (_tiktikTerminal.CheckTerminalPath())
                {
                    var terminalPath = _tiktikTerminal.BuildTerminalPath(data.Server, data.Login);

                    var checkResult = _terminalService.CheckTerminal(data.Server, data.Login, terminalPath);
                    if (checkResult == null) // Terminal not running
                    {
                        // Check terminal is running but system doesn't log it. 
                        var processes = Process.GetProcessesByName("terminal64");
                        var runningProcess = processes.Where(p => p.MainModule.FileName.StartsWith(terminalPath)).FirstOrDefault();
                        if (runningProcess != null)
                        {
                            this._tikTikProcess.KillProcess(runningProcess.Id);
                        }

                        // Start terminal
                        var terminalInfo = new Terminal()
                        {
                            Login = data.Login,
                            Server = data.Server,
                            TerminalPath = terminalPath,
                            Port = _tiktikTerminal.Port,
                            StateId = (int)TerminalStates.Starting,
                            SystemRequest = data.IsSystemRequest,
                            UserRequest = !data.IsSystemRequest
                        };

                        terminalInfo = _terminalService.Add(terminalInfo);

                        // Create terminal configuration
                        _tiktikTerminal.BuildConfiguration(data.Server, data.Login, data.Password);
                        var configPath = _tiktikTerminal.BuildConfigPath(terminalPath);
                        int processId = 0;
                        int retries = 0;
                        while (retries <= this._maxRetriesTime)
                        {
                            try
                            {
                                processId = _tikTikProcess.StartTerminal(terminalPath, configPath, _tiktikTerminal.MtApiEA);
                                terminalInfo.ProcessId = processId;
                                terminalInfo.StateId = (int)TerminalStates.Running;
                                terminalInfo.PopulateUpdated(0);
                                _terminalService.Update(terminalInfo);
                                _log.Debug($"Terminal is running with process id {processId} on port {_tiktikTerminal.Port}");
                                break;
                            }
                            catch (Exception ex)
                            {
                                retries++;
                                if (retries <= this._maxRetriesTime)
                                {
                                    _log.Error(ex.Message, ex);
                                    _log.Debug($"Retries {retries}");
                                    Thread.Sleep(1000);
                                }
                                else
                                {
                                    terminalInfo.StateId = (int)TerminalStates.Error;
                                    terminalInfo.PopulateUpdated(0);
                                    terminalInfo.Error = ex.Message.Substring(0, 255);
                                    _terminalService.Update(terminalInfo);
                                    throw ex;
                                }
                            }
                        }
                    }
                    else // Terminal is running
                    {
                        _log.Debug($"Terminal is running with process id {checkResult.ProcessId} with port {checkResult.Port}");
                        //TODO:
                    }
                }
            }
            catch (Exception ex)
            {
                this._log.Error(ex.Message, ex);
                throw ex;
            }
        }
```
- Create an order and submit the order within the transaction scope, as shown in the following sample code.
* Recevice Message:
``` js
  //Recevice Message
   public void ReceiveMessage()
        {
            using (MessageQueue myQueue = new MessageQueue(this._runningQueueName))
            {
                int retries = 0;
                string data = string.Empty;
                this._log.Debug($"Start message queue: {_runningQueueName}");
                myQueue.Formatter = new XmlMessageFormatter(new String[] { "System.String,mscorlib" });
                myQueue.PeekCompleted += (s, e) =>
                {
                    MessageQueueTransaction myTransaction = new MessageQueueTransaction();
                    try
                    {
                        // Begin the transaction.
                        myTransaction.Begin();

                        // Receive the message. 
                        Message myMessage = myQueue.Receive(myTransaction);

                        data = (string)myMessage.Body;
                        // Display message information.
                        this._log.Debug($"Receive message: {data}");
                        this.ProcessMessage(data);

                        // Commit the transaction.
                        myTransaction.Commit();

                    }
                    catch (Exception ex)
                    {
                        retries++;
                        if (retries >= this._maxRetriesTime)
                        {
                            retries = 0;

                            this._log.Error($"Process data error: {data} \n{ex.GetAllExceptionMessagesAsSingleString()}", ex);
                            myTransaction.Commit();
                        }
                        else
                        {
                            // Roll back the transaction.
                            this._log.Debug($"Retry {retries} -  process message with data {data}");
                            myTransaction.Abort();
                            Thread.Sleep(3000);

                        }
                    }
                    myQueue.BeginPeek();
                };
                myQueue.BeginPeek();
            }
        }
```
* Send Message:
``` js
  //Send Messaage
   public void SendMessage(RestartRunningProcessorDto processorDto)
        {
            using (MessageQueue myQueue = new MessageQueue(this._queueName))
            {
                var transaction = new MessageQueueTransaction();
                try
                {
                    string message = JsonConvert.SerializeObject(processorDto);
                    transaction.Begin();
                    myQueue.Send(message, transaction);
                    transaction.Commit();
                }
                catch (Exception ex)
                {
                    transaction.Abort();
                }
            }
        }
```
### Running
``` js
1. Right click the project
2. Select Open folder in file explorer
3. Select the Bin folder
4. Select Folder Debug
5. Running File exe (Example: Client.exe)
```
