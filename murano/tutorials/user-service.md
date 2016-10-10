---
# User Management
---

* [Overview](#head_overview)
* [Tutorial example in scripting system](#head_tutorial_example)
* [Reference for example](#head_reference)

## <span id="head_overview">Overview</span>
This system is to manage users under the Murano solution. It supports user authentication, role-based access control, and storage per user.

#### User Authentication
* Basic (email & password)
* Token based
* Social auth

#### User Permission
We support user permission based on [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control).
In our system, there are three concrete elements: &rsquo;role&rsquo;, &rsquo;user&rsquo;, and &rsquo;endpoint&rsquo;. According to RBAC, we can control a user&rsquo;s access to endpoint with the concept below:

1. Assign endpoint to role
2. Assign role to user
3. User can access the endpoint of his role

<div align="center">
  <img src="https://github.com/unahouExosite/docs/blob/MUR-298/murano/assets/usm_rbac_1.png"><br>
</div>

However, URL can be different in variable practically. For example, &rsquo;*device/1/info*&rsquo; and &rsquo;*device/2/info*&rsquo; are different literally but can be considered the same endpoint. We don&rsquo;t want to create two endpoints for such a difference. That is why we add the fourth element called &rsquo;parameter&rsquo;.

&rsquo;Parameter&rsquo; represents a specific resource by name and value. We mark it like {&lt;parameter_name&gt;} in endpoint. Here we only need to create one endpoint &rsquo;device/{rid}/info&rsquo; for &rsquo;*device/1/info*&rsquo; and &rsquo;*device/2/info*&rsquo;.
When we grant user permission, we need to specify parameters for each role assignment.

<div align="center">
  <img src="https://github.com/unahouExosite/docs/blob/MUR-298/murano/assets/usm_rbac_2.png"><br>
</div>

If we want to grant &rsquo;UserA&rsquo; access to &rsquo;*device/1/info*&rsquo;, we can:

1. Create a role called &rsquo;Viewer&rsquo;.
2. Add parameter definition &rsquo;rid&rsquo; to role &rsquo;Viewer&rsquo;, which means parameter &rsquo;rid&rsquo; is available in role &rsquo;Viewer&rsquo;.
3. Assign endpoint &rsquo;*device/{rid}/info*&rsquo; to role &rsquo;Viewer&rsquo;.
4. Assign role &rsquo;Viewer&rsquo; with parameter &rsquo;rid&rsquo;(name) = 1(value) to &rsquo;UserA&rsquo;.
5. Now UserA is allowed access to &rsquo;*device/1/info*&rsquo; when we check his permission.

#### Storage Per User
We provide storage per user which can store data by key-value format. Since a user&rsquo;s properties are only email and name, we can put more individual information in storage (for example, address, birthday, etc.).


<br><br><br><br>

## <span id="head_tutorial_example">Tutorial Example in Scripting System</span>

Assume we are running a parking application.
There are two major roles in our system. One: the driver wants to park his vehicle. Two: the parking lot/garage charges drivers for parking.

To separate their permission, we should create at least two roles.
###### <span id="eg_createRole"></span>
```lua
-- Create role 'vehicle' for driver
local role_vehicle = {
    ["role_id"] = "vehicle"
}
User.createRole(role_vehicle)

-- Create role 'parking_area'
local role_parking_area = {
    ["role_id"] = "parking_area"
}
User.createRole(role_parking_area)
```

Suppose we have two users:

**User\_Parking\_Area** is in role &rsquo;parking_area&rsquo; and has unique ID = **1**.

###### <span id="eg_createUser"></span>
```lua
-- Create User_Parking_Area
local new_user = {
	["name"] = "User_Parking_Area",
	["email"] = "demo_parking_area@exosite.com",
	["password"] = "demo777"
}
local activation_code = User.createUser(new_user)
```

###### <span id="eg_activateUser"></span>

```lua
-- Need to activate user after creating
local parameters = {
	["code"] = activation_code
}
User.activateUser(parameters)

```

**User_Vehicle** is in role &rsquo;vehicle&rsquo; and has unique ID = **2**.

```lua
-- Create User_Vehicle
local new_user = {
	["name"] = "User_Vehicle",
	["email"] = "demo_vehicle@exosite.com",
	["password"] = "demo777"
}
local activation_code = User.createUser(new_user)

-- Activate User_Vehicle
local parameters = {
	["code"] = activation_code
}
User.activateUser(parameters)

```

Each vehicle has its own license plate number; thus we store info in user storage.

###### <span id="eg_createUserData"></span>
```lua
--- Store User_Vehicle's license plate number
local parameters = {
    [“id”] = 2, -- User ID of User_Vehicle
    [“license_plate_number”] = “QA-7712”
}
User.createUserData(parameters) -- We store value by creating since 'license_plate_number' is not existing in storage of User_Vehicle.

-- Initialize User_Vehicle's parking start time
local parameters = {
    [“id”] = 2, -- User ID of User_Vehicle
    [“parking_start_time”] = “0000-00-00 00:00:00” -- current parking start time
}
User.createUserData(parameters)
```
### Scenario: List Control
First, we support an endpoint for parking areas to manage their parking spaces. This endpoint is expected to list unique IDs of parking spaces.

```lua
-- Create endpoint for listing parking spaces of parking area
local list_parking_space_endpoint = {
    [“method”] = “GET”,
    [“end_point”] = “list/{parkingAreaID}/parkingSpace” -- This endpoint contains a parameter name 'parkingAreaID'
}
User.createPermission(list_parking_space_endpoint)

-- Assign endpoint to role 'parking_area', so parking areas can access it.
local endpoints = {
    ["method"] = "GET",
    [“end_point”] = “list/{parkingAreaID}/parkingSpace”
}
User.addRolePerm({[“role_id”] = “parking_area”, [“body”] = endpoints})
```

To let **User\_Parking\_Area** access *'GET list/1/parkingSpace'*, we need to grant **User\_Parking\_Area** permisson on 'parkingAreaID' = 1 in role 'parking_area'.

```lua
-- We should add parameter definition before assigning roles with new parameter name.
local param_definitions = {
    {
        [“name”] = “parkingAreaID”
    }
}
User.addRoleParam({[“role_id”] = “parking_area”,  [“body”] = param_definitions})

-- Grant User_Parking_Area parameter 'parkingAreaID' = 1 in role 'parking_area'
local roles_assigned = {
    {
        [“role_id”] = “parking_area”,
        [“parameters”] = {
            {
                [“name”] = “parkingAreaID”,
                [“value”] = 1
            }
        }
    }
}
User.assignUser({[“id”] = 1, [“roles”] = roles_assigned})
```
Now **User\_Parking\_Area** can access *'GET list/1/parkingSpace'*.

Next, we are going to make the list returned in response.

Assume there are two parking spaces in **User\_Parking\_Area**. Each parking space has a device RID. Devices can detect the status of a parking space. We can make **User\_Parking\_Area** have device RIDs by assigning roles. 

```lua
-- We should add parameter definition before assigning roles with new parameter name.
local param_definitions = {
    [“role_id”] = “parking_area”,
    [“body”] = {
        {
            [“name”] = “spaceRID” -- device RID
        }
    }
}
User.addRoleParam(param_definitions)

-- Grant User_Parking_Area access to his parking space RIDs
local roles_assigned = {
    {
        [“role_id”] = “parking_area”,
        [“parameters”] = {
            {
                [“name”] = “spaceRID”,
                [“value”] = “d2343hbcc1232sweee12” -- first parking space RID
             },
             {
                [“name”] = “spaceRID”,
                [“value”] = “a34feh709a234e232xd21” -- second parking space RID
             }
        }
    }
}
User.assignUser({[“id”] = 1, [“roles”] = roles_assigned})
```

After roles assignment, we can return a paginated list of &rsquo;spaceRID&rsquo; when **User\_Parking\_Area** accesses *'GET list/1/parkingSpace'*.

###### <span id="eg_listUserRoleParamValues"></span>
```lua
local parameters = {
    [“id”] = 1, -- User ID of User_Parking_Area
    [“role_id”] = “parking_area”,
    [“parameter_name”] = “spaceRID”,
    [“offset”] = 0, -- offset for pagination
    [“limit”] = 10 -- limit for pagination
}
local result = User.listUserRoleParamValues(parameters)
response.message = result.items -- return the list of RID
```

### Scenario: Endpoint Access Control
Second, we support an endpoint for drivers to look for a vacant parking space. Drivers can query a number of vacancy in every parking area while each parking area is restricted to his own.

###### <span id="eg_createPermission"></span>
```lua
-- Create endpoint for querying vacant parking space
local available_space_endpoint = {
    [“method”] = “GET”,
    [“end_point”] = “query/{parkingAreaID}/availableSpace” -- The endpoint contains a parameter name 'parkingAreaID'.
}
User.createPermission(available_space_endpoint)
```
###### <span id="eg_addRolePerm"></span>
```lua
-- Assign the endpoint to roles 'vehicle' and 'parking_area', so drivers and parking areas can access the endpoint.
local endpoints = {
    {
        [“method”] = “GET”,
        [“end_point”] = “query/{parkingAreaID}/availableSpace”
    }
}
-- Let drivers access this endpoint
User.addRolePerm({[“role_id”] = “vehicle”, [“body”] = endpoints})
-- Let parking areas access this endpoint
User.addRolePerm({[“role_id”] = “parking_area”, [“body”] = endpoints})
```

To let **User\_Parking\_Area** access *'GET query/1/availableSpace'*, **User\_Parking\_Area** should have parameter 'parkingAreaID' = 1 in role 'parking_area'. Since he has been assigned it before, there is no need to assign again.

To let **User\_Vehicle** access *'GET query/\*/availableSpace'*, we should grant **User\_Vehicle** permission on 'parkingAreaID' = * in role 'vehicle'.

###### <span id="eg_addRoleParam"></span>
```lua
-- To role 'vehicle', parameter name 'parkingAreaID' is new. Before assigning roles with new parameter name, we need to add parameter definition.
local param_definitions = {
    {
        [“name”] = “parkingAreaID”
    }
}
User.addRoleParam({[“role_id”] = “vehicle”,  [“body”] = param_definitions})
```

###### <span id="eg_assignUser"></span>
```lua
-- Grant User_Vehicle all value of parkingAreaID in role 'vehicle'
local roles_assigned = {
    {
        [“role_id”] = “vehicle”,
        [“parameters”] = {
            {
                [“name”] = “parkingAreaID”,
                [“value”] = {
                    [“type”] = “wildcard” -- this format represents all values
                }
            }
        }
    }
}
User.assignUser({[“id”] = 2, [“roles”] = roles_assigned})
```

Now we can prepare the number returned in response.

We already know there are two parking spaces in **User\_Parking\_Area**. We can store info in storage.
```lua
-- Add number of vacancy info for User_Parking_Area
local updating_data = {
    [“id”] = 1, -- User ID of User_Parking_Area
    [“number_of_vacancy”] = 2, -- Assuming currently there is no vehicle parked in User_Parking_Area.
}
User.createUserData(updating_data) -- We store value by creating since 'number_of_vacancy' is not existing in storage of User_Parking_Area.
```

When permitted user accesses *'GET  query/1/availableSpace'*, we can return the number.
```lua
-- Get number of vanacy of User_Parking_Area
local parameters = {
    [“id”] = 1, -- User ID of User_Parking_Area
    [“key”] = "number_of_vacancy",
}
local number = User.getUserData(parameters)
response.message = number
```


The underground process of permission check when the user accesses endpoint can be replicated as follows:
###### <span id="eg_hasUserPerm"></span>
```lua
-- Check if User_Vehicle can access GET query/1/availableSpace.
local check_permission = {
    [“id”] = 2, -- User ID of User_Vehicle
    [“perm_id”] = “GET%2query%2F%7BparkingAreaID%7D%2FavailableSpace”, -- urlencode("GET/query/{parkingAreaID}/availableSpace")
    [“parameters”] = {
        “parkingAreaID::1” -- Specifies value 1 for parameter 'parkingAreaID' in endpoint
    }
}
local result = User.hasUserPerm(check_permission)
```

Because **User\_Vehicle** has been assigned with all values of &rsquo;parkingAreaID&rsquo; in role &rsquo;vehicle&rsquo;, variable &rsquo;result&rsquo; is expected to be &rsquo;OK&rsquo;.

```lua
-- Check if User_Parking_Area can access 'GET query/1/availableSpace'.
local check_param = {
    [“id”] = 1, -- User ID of User_Parking_Area
    [“perm_id”] = “GET%2query%2F%7BparkingAreaID%7D%2FavailableSpace”, -- urlencode("GET/query/{parkingAreaID}/availableSpace")
    [“parameters”] = {
        “parkingAreaID::1” -- Specifies value 1 for parameter 'parkingAreaID' in endpoint
    }
}
local result = User.hasUserPerm(check_param)
```

Because **User\_Parking\_Area** has been assigned with &rsquo;parkingAreaID = 1&rsquo; in role &rsquo;parking_area&rsquo;, variable &rsquo;result&rsquo; is expected to be &rsquo;OK&rsquo;.




### Scenario: Application of User-storage and Endpoint-access-control

We also support an endpoint for parking areas to query info of a vehicle, such as parking time or license plate number. Each parking area can only see info of vehicles parked in its spaces.

```lua
-- Create endpoint 'GET query/{parkingAreaID}/parkingVehicle/{vehicleID}/info'
local vehicle_info_endpoint = {
    ["method"] = "GET",
    ["end_point"] = "query/{parkingAreaID}/parkingVehicle/{vehicleID}/info"
}
User.createPermission(vehicle_info_endpoint)

-- Assign endpoint to role 'parking_area', so parking areas can access this endpoint.
local endpoints_assigned = {
    {
        ["method"] = “GET”,
        ["end_point"] = "query/{parkingAreaID}/parkingVehicle/{vehicleID}/info" -- This endpoint contains a parameter name 'vehicleID'
    }
}
User.addRolePerm({["role_id"] = "parking_area", ["body"] = endpoints_assigned})

-- Add parameter definition 'vehicleID' to role 'parking_area' for assigning role 'parking_area' with parameter 'vehicleID'
local param_definitions = {
    {
        ["name"] = "vehicleID"
    }
}
User.addRoleParam({["role_id"] = "parking_area",  ["body"] = param_definitions})
```

When **User\_Vehicle** parks in **User\_Parking\_Area** (detected by device), **User\_Parking\_Area** should have the right to see info of **User\_Vehicle**.

```lua
-- Since User_Parking_Area already has 'parkingAreaID' = 1 in role 'parking_area', we only need to assign role 'parking_area' with extra parameter 'vehicleID' = 2 to User_Parking_Area.
local roles_assigned = {
    {
        ["role_id"] = "parking_area",
        ["parameters"] = {
            {
                ["name"] = "vehicleID",
                ["value"] = 2 -- User ID of User_Vehicle
            }
        }
    }
}
User.assignUser({["id"] = 1, ["roles"] = roles_assigned})
```

###### <span id="eg_updateUserData"></span>
```lua
-- Add parking info for User_Vehicle
local updating_data = {
    ["id"] = 2, -- User ID of User_Vehicle
    ["parking_start_time"] = “2016-08-30 08:10:10”
}
User.updateUserData(updating_data)
```

For info of the space **User\_Vehicle** is parking at, we can create another parameter 'parkingSpaceRID' to store relationship by role assignment.

```lua
-- Create parameter definition for new parameter.
local param_definitions = {
    ["role_id"] = "vehicle",
    ["body"] = {
        {
            ["name"] = "parkingSpaceRID" -- device RID of parking space
        }
    }
}
User.addRoleParam(param_definitions)
-- User_Vehicle is parking at space 'd2343hbcc1232sweee1'.
local roles_assigned = {
    {
        ["role_id"] = "vehicle",
        ["parameters"] = {
            {
                ["name"] = "parkingSpaceRID",
                ["value"] = "d2343hbcc1232sweee1" -- device RID of parking space
            }
        }
    }
}
User.assignUser({["id"] = 2, ["roles"] = roles_assigned})
```

Because parking space &rsquo;d2343hbcc1232sweee1&rsquo; is occupied, we should also update the number of vacancy of **User\_Parking\_Area**.
```lua
-- Get current number of vacancy of User_Parking_Area
local parameters = {
    ["id"] = 1,
    ["key"] = "number_of_vacancy"
}
local latest_number = User.getUserData(parameters)

-- Update number of vacancy to latest
local updating_data = {
    ["id"] = 1,
    ["number_of_vacancy"] = latest_number - 1 -- User_Vehicle just occupied one parking space.
}
User.updateUserData(updating_data)
```

Now **User\_Parking\_Area** can access *'GET query/1/parkingVehicle/2/info'* to get parking info of **User\_Vehicle**.

###### <span id="eg_getUserData"></span>
```lua
-- Get parking info of User_Vehicle

local parameters = {
    ["id"] = 2,
    ["key"] = "parking_start_time"
}
local parking_start_time = User.getUserData(parameters)


local parameters = {
    ["id"] = 2,
    ["key"] = "license_plate_number"
}
local license_plate_number = User.getUserData(parameters)

-- Find parking space User_Vehicle is parking at

local parameters = {
    ["id"] = 2, -- User ID of User_Vehicle
    ["role_id"] = "vehicle",
    ["parameter_name"] = "parkingSpaceRID"
}
local values = User.listUserRoleParamValues(parameters)
local parking_space_rid = values[1]

-- response with parking info
response.message = {
    ["parking_start_time"] = parking_start_time,
    ["license_plate_number"] = license_plate_number,
    ["parking_space_rid"] = parking_space_rid
}
```

When **User\_Vehicle** is going to leave, **User\_Parking\_Area** can charge him according to the parking time.


After **User\_Vehicle** leaves, we remove **User\_Vehicle** from the parking list of **User\_Parking\_Area** and update relevant info.

```lua
local roles_removed = {
    ["id"] = 1, -- User ID of User_Parking_Area
    ["role_id"] = "parking_area",
    ["parameter_name"] = "vehicleID",
    ["parameter_value"] = 2 -- User ID of User_Vehicle
}
User.deassignUserParam(roles_removed)

-- Also remove relationship between User_Vehicle and parking space
local roles_removed = {
    ["id"] = 2, -- User ID of User_Vehicle
    ["role_id"] = "vehicle",
    ["parameter_name"] = "parkingSpaceRID",
    ["parameter_value"] = "d2343hbcc1232sweee1"
}
User.deassignUserParam(roles_removed)

-- Update parking info of User_Vehicle
local updating_data = {
    ["id"] = 2, -- User ID of User_Vehicle
    ["parking_start_time"] = “0000-00-00 00:00:00”
}
User.updateUserData(updating_data)

-- Update number of vacancy of User_Parking_Area
local parameters = {
    ["id"] = 1,
    ["key"] = "number_of_vacancy"
}
local latest_number = User.getUserData(parameters)
local updating_data = {
    ["id"] = 1,
    ["number_of_vacancy"] = latest_number + 1 -- User_Vehicle just left.
}
User.updateUserData(updating_data)
```

**User\_Parking\_Area** cannot access *'GET query/1/parkingVehicle/2/info'* any longer since **User\_Vehicle** is not in his parking list.

```lua
-- Underground process of checking if User_Parking_Area can access 'GET query/1/parkingVehicle/2/info'.

local check_permission = {
    ["id"] = 1, -- User ID of User_Parking_Area
    ["perm_id"] = "GET%2Fquery%2F%7BparkingAreaID%7D%2FparkingVehicle%2F%7BvehicleID%7D%2Finfo", -- urlencode("GET/query/{parkingAreaID}/parkingVehicle/{vehicleID}/info")
    ["parameters"] = {
        "parkingAreaID::1", -- Specifies value 1 for parameter 'parkingAreaID' in endpoint
        "vehicleID::2" -- Specifies value 2 for parameter 'vehicleID' in endpoint
    }
}
local result = User.hasUserPerm(check_permission)
```

Because currently **User\_Parking\_Area** does not have parameter &rsquo;vehicleID&rsquo; = 2 in role &rsquo;parking_area&rsquo;, it is expected to get result.status == 403.


<br><br><br><br>

## <span id="head_reference">Reference for Example</span>
* RBAC
   * User
      * [User.createUser()](#eg_createUser)
      * [User.activateUser()](#eg_activateUser)
   * Role
      * [User.createRole()](#eg_createRole)
      * [User.addRoleParam()](#eg_caddRoleParam)
   * Endpoint
      * [User.createPermission()](#eg_createPermission)
   * Role-Endpoint Relation
      * [User.addRolePerm()](#eg_addRolePerm)
   * Role-User Relation
      * [User.assignUser()](#eg_assignUser)
      * [User.deassignUserParam()](#eg_deassignUserParam)
      * [User.hasUserPerm()](#eg_hasUserPerm)
      * [User.listUserRoleParamValues()](#eg_listUserRoleParamValues)

* User Storage
   * [User.createUserData()](#eg_createUserData)
   * [User.updateUserData()](#eg_updateUserData)
   * [User.getUserData()](#eg_getUserData)











<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>
