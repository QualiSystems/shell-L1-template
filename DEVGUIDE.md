# L1 Driver Devguide
This guide is a short review of the python L1 driver, description of the commands with examples.

## L1 Driver Review

L1 driver is a self executable application which is running by CloudShell with a specific port like an argument "L1_DRIVER.exe 4000". 
It starts listening to a specified port and waits for a connection from the CloudShell. 
CloudShell communicate with the driver using XML commands.

**Request command example**
```xml
<Commands xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.qualisystems.com/ResourceManagement/DriverCommands.xsd">
    <Command CommandName="Login" CommandId="ae39207a-a65c-40e9-bec3-4996ddcef727">
        <Parameters xsi:type="LoginCommandParameters">
            <Address>192.168.42.240</Address>
            <User>admin</User>
            <Password>admin</Password>
        </Parameters>
    </Command>
</Commands>
```
**Response example**
```xml
<Responses xmlns="http://schemas.qualisystems.com/ResourceManagement/DriverCommandResult.xsd" Success="true">
    <CommandResponse CommandName="Login" Success="true" CommandId="ae39207a-a65c-40e9-bec3-4996ddcef727">
        <Error/>
        <Log/>
        <Timestamp>14.12.2016 11:41:52</Timestamp>
        <ResponseInfo/>
    </CommandResponse>
    <ErrorCode/>
    <Log>No need for Sync</Log>
</Responses>
```

### Driver commands

#### Login
Opens new session to the device, called before each command tuple

**Attributes**
* *Address* - IP address of the device
*  *User* - Login username
*  *Password* - Password

####*Example of implementation*
```python
    def login(self, address, username, password):
        # Define session attributes
        self._cli_handler.define_session_attributes(address, username, password)
        
        # Obtain cli session
        with self._cli_handler.default_mode_service() as session:
            # Executing simple command
            device_info = session.send_command('show version')
            self._logger.info(device_info)
```


#### GetStateId
Checks synchronization state with CloudShell, if state_id is different, then CloudShell and device is not synchronized. Return -1 if not used.

####*Example of implementation*

```python
    def get_state_id(self):
        """
        Check if CS synchronized with the device.
        :return: Synchronization ID, GetStateIdResponseInfo(-1) if not used
        :rtype: cloudshell.layer_one.core.response.response_info.GetStateIdResponseInfo
        :raises Exception: if command failed
        """
        # Obtain cli session
        with self._cli_handler.default_mode_service() as session:
            # Execute command
            chassis_name = session.send_command('show chassis name')
            return chassis_name
```


#### SetStateId
Set synchronization state id to the device, it calls after any change done

**Attributes**
* *StateId* - Id

####*Example of implementation*
```python
    def set_state_id(self, state_id):
        """
        Set synchronization state id to the device, called after Autoload or SyncFomDevice commands
        :param state_id: synchronization ID
        :type state_id: str
        :return: None
        :raises Exception: if command failed
        """
        # Obtain cli session
        with self._cli_handler.config_mode_service() as session:
            # Execute command
            session.send_command('set chassis name {}'.format(state_id))
```
#### GetResourceDescription
Auto-load function to retrieve all information from the device and build device structure

**Attributes**
* *Address* - resource address, '192.168.42.240'

####*Example of implementation*
```python
    def get_resource_description(self, address):
        """
        Auto-load function to retrieve all information from the device
        :param address: resource address, '192.168.42.240'
        :type address: str
        :return: resource description
        :rtype: cloudshell.layer_one.core.response.response_info.ResourceDescriptionResponseInfo
        :raises cloudshell.layer_one.core.layer_one_driver_exception.LayerOneDriverException: Layer one exception.
        """
        from cloudshell.layer_one.core.response.resource_info.entities.chassis import Chassis
        from cloudshell.layer_one.core.response.resource_info.entities.blade import Blade
        from cloudshell.layer_one.core.response.resource_info.entities.port import Port

        chassis_resource_id = chassis_info.get_id()
        chassis_address = chassis_info.get_address()
        chassis_model_name = "{{ cookiecutter.model_name }} Chassis"
        chassis_serial_number = chassis_info.get_serial_number()
        chassis = Chassis(resource_id, address, model_name, serial_number)

        blade_resource_id = blade_info.get_id()
        blade_model_name = 'Generic L1 Module'
        blade_serial_number = blade_info.get_serial_number()
        blade.set_parent_resource(chassis)

        port_id = port_info.get_id()
        port_serial_number = port_info.get_serial_number()
        port = Port(port_id, 'Generic L1 Port', port_serial_number)
        port.set_parent_resource(blade)

        return ResourceDescriptionResponseInfo([chassis])
```

#### MapBidi
Create a bidirectional connection between source and destination ports

**Attributes**
* *MapPort_A* - src port address, '192.168.42.240/1/21'
* *MapPort_B* - dst port address, '192.168.42.240/1/22'

####*Example of implementation*
```python
    def map_bidi(self, src_port, dst_port):
        """
        Create a bidirectional connection between source and destination ports
        :param src_port: src port address, '192.168.42.240/1/21'
        :type src_port: str
        :param dst_port: dst port address, '192.168.42.240/1/22'
        :type dst_port: str
        :return: None
        :raises Exception: if command failed
        """
        with self._cli_handler.config_mode_service() as session:
            session.send_command('map bidir {0} {1}'.format(convert_port(src_port), convert_port(dst_port)))
```

#### MapUni
Unidirectional mapping of ports, one src_port and multiple dst_ports, 

**Attributes**
* *SrcPort* - src port address, '192.168.42.240/1/21'
* *DstPort* - dst port address, '192.168.42.240/1/22'
* *DstPort* - dst port address, '192.168.42.240/1/23'


####*Example of implementation*
```python
    def map_uni(self, src_port, dst_ports):
        """
        Unidirectional mapping of two ports
        :param src_port: src port address, '192.168.42.240/1/21'
        :type src_port: str
        :param dst_ports: list of dst ports addresses, ['192.168.42.240/1/22', '192.168.42.240/1/23']
        :type dst_ports: list
        :return: None
        :raises Exception: if command failed
        """
        with self._cli_handler.config_mode_service() as session:
            for dst_port in dst_ports:
                session.send_command('map {0} also-to {1}'.format(convert_port(src_port), convert_port(dst_port)))
            
```

#### MapClear
Clear mapping on the port, for multiple ports
**Attributes**
* *MapPort* - port adress, '192.168.42.240/1/21' 
* *MapPort* - port adress, '192.168.42.240/1/22' 

####*Example of implementation*
```python
    def map_clear(self, ports):
        """
        Remove simplex/multi-cast/duplex connection ending on the destination port
        :param ports: ports, ['192.168.42.240/1/21', '192.168.42.240/1/22']
        :type ports: list
        :return: None
        :raises Exception: if command failed
        """
        with self._cli_handler.config_mode_service() as session:
            mapping_actions = MappingActions(session, self._logger)
            _ports = [convert_port(port) for port in ports]
            mapping_actions.map_clear(_ports)
```

#### MapClearTo
Remove unidirectional mapping for one src_port and multiple dst_ports

**Attributes**
* *SrcPort* - src port address, '192.168.42.240/1/21'
* *DstPort* - dst port address, '192.168.42.240/1/22'
* *DstPort* - dst port address, '192.168.42.240/1/23'

####*Example of implementation*
```python
    def map_clear_to(self, src_port, dst_ports):
        """
        Remove simplex/multi-cast/duplex connection ending on the destination port
        :param src_port: src port address, '192.168.42.240/1/21'
        :type src_port: str
        :param dst_ports: list of dst ports addresses, ['192.168.42.240/1/21', '192.168.42.240/1/22']
        :type dst_ports: list
        :return: None
        :raises Exception: if command failed
        """
        with self._cli_handler.config_mode_service() as session:
            mapping_actions = MappingActions(session, self._logger)
            _src_port = Address.from_cs_address(src_port).build_str()
            _dst_ports = [Address.from_cs_address(port).build_str() for port in dst_ports]
            mapping_actions.map_clear_to(_src_port, _dst_ports)
```


#### GetAttributeValue
Retrieve attribute value from the device for a specific resource

**Attributes**
* *Address* - Resource address, '10.11.178.35/2/1'
* *Attribute* - Attribute name, 'Model Name'

####*Example of implementation*
```python
    def get_attribute_value(self, cs_address, attribute_name):
        """
        Retrieve attribute value from the device
        :param cs_address: address, '192.168.42.240/1/21'
        :type cs_address: str
        :param attribute_name: attribute name, "Port Speed"
        :type attribute_name: str
        :return: attribute value
        :rtype: cloudshell.layer_one.core.response.response_info.AttributeValueResponseInfo
        :raises Exception: if command failed
        """
        with self._cli_handler.config_mode_service() as session:
            command = AttributeCommandFactory.get_attribute_command(cs_address, attribute_name)
            value = session.send_command(command)
            return AttributeValueResponseInfo(value)
```

#### SetAttributeValue
Set attribute value to the device

**Attribute**
* *Address* - resource address, '10.11.178.35/4'
* *Attribute* - attribute name, 'Toggle Rate'
* *Value* - attribute value, '9999'

####*Example of implementation*
```python
    def set_attribute_value(self, cs_address, attribute_name, attribute_value):
        """
        Set attribute value to the device
        :param cs_address: address, '192.168.42.240/1/21'
        :type cs_address: str
        :param attribute_name: attribute name, "Port Speed"
        :type attribute_name: str
        :param attribute_value: value, "10000"
        :type attribute_value: str
        :return: attribute value
        :rtype: cloudshell.layer_one.core.response.response_info.AttributeValueResponseInfo
        :raises Exception: if command failed
        """
        with self._cli_handler.config_mode_service() as session:
            command = AttributeCommandFactory.set_attribute_command(cs_address, attribute_name, attribute_value)
            session.send_command(command)
            return AttributeValueResponseInfo(attribute_value)

```

### Basic modules and classes
* **Module main** *main.py* - Enter point of the driver, initialize and start driver.
* **Class DriverCommands** *driver_commands.py* - Main driver class, implements [DriverCommandsInterface](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/driver_commands_interface.py), one method for each driver command.
* **Class [CommandExecutor](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/command_executor.py)** - Extract attributes from the command request and execute specific method of the DriverCommands class implemented for this specific driver command.
* **Class [DriverListener](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/driver_listener.py)** - Listen for the new connections from the CloudShell.
* **Class [ConnectionHandler](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/connection_handler.py)** - Handle new connection, read requests and send responses.

### How it works
1. CloudShell starts driver executable with port value as an argument ```L1_DRIVER.exe 4000```. It unpack interpreter with libraries into local virtual environment, then starts ```main.py``` with arguments.
2. Module ```main``` reads runtime configuration, initializes instances of loggers, implemented DriverCommands class, [CommandExecutor](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/command_executor.py), and [DriverListener](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/driver_listener.py), then start listening new connections. 
3. When driver started, CloudShell initiate new connection to the driver and send XML command tuple. Each command tuple has Login command at the beginning.
4. New connections handles by [ConnectionHandler](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/connection_handler.py).
    * Read request data from the socket
    * Parse request data by [RequestParser](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/request/requests_parser.py) and creates list of [CommandRequest](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/request/command_request.py) objects.
    * Execute list of [CommandRequest](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/request/command_request.py) by [CommandExecutor](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/command_executor.py)  
5. [CommandExecutor](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/command_executor.py) extract command_name and attributes for each [CommandRequest](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/request/command_request.py) instance, call method of implemented DriverCommands class defined for this command_name.
    * For each [CommandRequest](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/request/command_request.py) [CommandExecutor](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/command_executor.py) creates [CommandResponse](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/response/command_response.py) instance, which uses for generating response.
6. [ResponseInfo](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/response/response_info.py) has response_info attribute, it can be defined by [ResponseInfo](https://github.com/QualiSystems/cloudshell-L1-networking-core/blob/refactoring/cloudshell/layer_one/core/response/response_info.py) instance, which can be returned from methods of DriverCommands class.
 
### CLI Usage

#### Example of DriverCommands class with CLI usage [DriverCommands](https://github.com/QualiSystems/cloudshell-L1-mrv/blob/model_fixes/mrv/driver_commands.py)