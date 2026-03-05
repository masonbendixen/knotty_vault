---
fileClass: Project
Category: Planning
Status: Active
tags:
  - website
  - client
  - server
---
# Overview
- I have got the server side largely done for CRUD support. The client side has a few of the controls wrapped up.
- Modify the metadata in the client to match that on the server
- Modify the controls that I have to support the new metadata
- Finish the aggregate control that does the create / update stuff
- Write an aggregate control that displays the aggregate control for a whole row of data and supports sending the create / update to the server
- Write some simple display controls to show data
- Write an aggregate control to show a row of data and support edit / delete
- Write a larger control that shows the rows of a table with pagination

# What I'm working on with status
- 8/6 Updating client
	- Currently src/app/shared/types/ServerDataInfo.ts
	```typescript
	export interface ServerDataInfo {
	  'data-name'?: string;      // The name for the data
	  label: string;            // The label to display for the textarea
	  hint?: string;             // Hint string for control
	  'place-holder'?: string;   // Place-holder string for control
	  regex?: string;            // Validation regular expression
	  'html-input-type'?: string; 
	  required?: boolean;        // Indicates whether the textarea is required
	  'max-length'?: number;     // The maximum number of characters allowed
	  'value': string;          // The default value of the textarea
	  rows?: number;             // The number of rows for the textarea
	}
	```
	- Change to ColumnDataInfo
```typescript
interface ColumnDataInfo {
    columnDataInfoId: number; // Primary key, not nullable
    tableName: string; // Foreign key, not nullable
    columnName: string; // Not nullable
    label?: string;
    hint?: string;
    placeHolder?: string;
    regex?: string;
    htmlInputType?: string;
    required?: boolean;
    maxLength?: number;
    defaultValue?: string;
    rows?: number;
}
```

- 8/7 Need to identify the REST APIs and JSON format here to figure out my client workflows. CRUD
	- /api/add_item POST
	```typescript
	export interface AddItemBody {
		table_name: string;
		value: any; // Object with key value pairs to insert into database
	}
	```
	- /api/add_item_fetch_primary_key POST
		- Returns a string with the primary key created
		- Takes an AddItemBody like add_item
	- /api/get_table_rows/\<string> - where string is table_name
		```typescript
		export interface DataResults {
			sortedColumnNames: string[]; // Keys in sorted order
			// Arrays of string arrays with values matching key order
			dataTable: string[][];  
		}
		```
		- Returns a DataResults
	- /api/get_rows_by_column/\<string>/\<string>/\<int>/\<int>/\<int>
		- Arguments are tableName, columnName, a "bool" for ascending (1 or 0), page size, page number
		- Returns a DataResult like get_table_rows
		- This function lets you sort by the given column and do pagination
	- /api/update_item POST
		```typescript
		export interface UpdateItemBody {
			table_name: string;
			value: any; // Object with key value pairs to insert into database
			column_name: string; // Name of column to update by
			column_value: string; // Name of column
		}
		```
		- Updates the key value pairs of the row(s) in table_name who have the designated column with the provided value.
	- /api/delete_item/\<string>/\<string>/\<string>
		- Parameters are table name, column name, and column value.
		- Removes the given row(s) who have a value in the specified column matching the specified value.
	- /api/get_db_schema
		- Returns a database schema in the given format:
		```typescript
export interface ForeignKeyInfo {
    column_name: string; // "column in this table"
    parent_column_name: string; // "column referenced in parent"
    parent_table_name: string; // "table referenced"
}	

export interface ColumnDataInfo {
    column_friendly_name: string; // Specified or generated
    column_name: string; //
    default_value: string; // Value to list on new item creation
    hint: string; // HTML hint string
    html_input_type: string; // text, long-text, bool, date
    label: string; // Value to use for the paired control label
    max_length: number; // Max characters allowed
    nullable: boolean; // Can this value be null
    place_holder: string; // HTML placeholder value
    primary_key: boolean; // True if this item is the primary key
    regex: string; // Pattern used to validate this expression
    required: boolean; // Maps to the HTML required field
    rows: string; // Number of rows for multi line controls
    type: string; // SQL type of the value
    unique: string; // True if this item is unique in the SQL sense
}

export interface TableSchema {
    columns: ColumnDataInfo[];
    description: string; // "..."
    foreign_keys: ForeignKeyInfo[];
    primary_key: string; // "..."
    table_friendly_name: string; // "..."
    table_name: string; // "..."
}

export interface DatabaseSchema {
    root_tables: string[]; // List of top level tables
    tables: TableSchema[];
}	
		```
	- Client needs a service that holds the DatabaseSchema. It should cache it. On site startup, it should send an async request and send a timer to start again if that one doesn't succeed and cancel after it sends it. On request, it should start another request if the data isn't present and then block on the result.
		- The service should have helpers to fetch the list of top level tables, list the TableSchema, and fetch a particular TableSchema. 
	- Need to modify the controls to take an optional value input param that is used to initially populate the control.
	- Need to modify the controls not to use htmlInputType for their HTML type. This is just for the composite control
	- Need to make a RowControl that takes an array of column names, an optional array of values, and then pulls the DatabaseSchemaService to send the optional value and ColumnDataInfo to all of the various controls that it makes on the fly. It also takes a bool that indicates update or create to determine if, on post, it does an update or a create. It should also have a back button to go back to where we came from. The control should also keep track of the value and column name of the primary key.
	- Need to make a ViewControl that takes a ColumnDataInfo and a value and displays the value appropriately.
	- Need to make a RowViewControl that takes an array of column names, an array of values, and then pulls the DatabaseSchemaService to send the value and ColumnDataInfo to the ViewControl. It will display all of the values as well as labels and buttons to edit / delete each row.
	- Need to make a TableViewControl that takes an array of colomn names, an array of arrays of values, and then pulls the DatabaseSchemaService to send the ColumnDataInfo array and each row to a different RowViewControl. Should also have controls to choose the number of rows, jump to page, next / prev, and first last page navigation. Might be able to use Lucas's table control for this.

- 8/11 What I'm working on
	- Added ForeignKeyInfo, TableSchema, and DatabaseSchema to the change request client_2_interfaces and sent it out for review.
	- Create DatabaseSchemaService that holds a DatabaseSchema
		- Branch client_3_database_schema
		- StartLazyLoad() - Called at site startup to begin an async operation to load this metadata from the server
			- Also starts a timer to try again if the first one fails.
		- WaitForMetadata() - Restarts an async operation to load the metadata in addition to any that might be pending.
		- Callback for async completion
			- Retry on failure
			- Cache metadata, set state for no more changes, cancel async requests on success
		- GetDatabaseSchema()
		- GetTableSchema(tableName: string)
		- Come up with a strategy for a test versus production version where the test version has default DatabaseSchema and the production version calls to the network. Maybe make these separate components that are used by the service and one is injected in production and one in test.
	- Making a new Angular environment
		- Currently we have production and development
		- These live under src/environments/environment.\<type>.ts
		- In angular.json, under build there are sections for both environment types and a fileReplacements section that causes which one is active to be copied to just environment.ts
		- We will now have:
			- production
			- development-mock
			- development-network
		- Create a network wrapper
			- ServerAccess is an interface
			- LocalServerAccess implements the interface locally
			- NetworkServerAccess implements the interface to call the network
			- SERVER_ACCESS_IMPLEMENTATION_TOKEN is an injection token to provide the correct implementation
			- ServerAccessProxy implements the ServerAccess interface and internally holds a ServerAccess reference that is conditionally bound based on environment to the correct SERVER_ACCESS_IMPLEMENTATION_TOKEN and ServerAccess implementation. Use conditional imports to do this at runtime.
			- SERVER_ACCESS_TOKEN is an injection token to inject ServerAccessProxy
			- Use file replacement in angular.json. Default to the local case but copy over for prod and the network development case.
			- portal / services
				- Create ServerAccess.ts that with a ServerAccessProxy class implementing ServerAccess that provides the SERVER_ACCESS_TOKEN and creates a SERVER_ACCESS_IMPLEMENTATION_TOKEN to defer to the implementation for
				- Under directory network_abstraction folder
				- Create ServerAccess.ts under network abstraction that reads the server name and port from the environment file and does the network access.
				- CreateServerAccess.local.ts under network abstraction that is file replaced in angular.json in the development-mock case that returns mocked data and provides SERVER_ACCESS_TOKEN
		- Use the network wrapper to implement DatabaseSchema.

- 8/12 What I'm working on
	- Created client_3_network branch
	- Created portal/services
	- Created portal/types
	- Created portal/types/ServerAccess.ts
		- Added all the methods
	- Created portal/services/ServerAccess.ts
	- Created portal/services/network_abstraction
	- Created portal/services/network_abstraction/ServerAccess.ts
		- Implemented this by calling to the assorted HTTP calls to the server
	- Created portal/services/network_abstraction/ServerAccess.mock.ts
	- ColumnDataInfo is messed up
		- A lot of the fields that aren't extra metadata are missing
		- There is a conflict between snake case and camel case
		- The bools and numbers are all strings and the bools are t or f
	- Here is the CoPilot query to generate the Metadata for the database:
	> Using the files inside the directory db_schema, each cpp file declares the metadata for a table. Can you generate JSON like in the database_rest_helper.h like the comment before DatabaseMetadata for the classes and people tables in db_schema using the constants in the headers to fill in the values? The friendly names and fields like that are defined in create_database.cpp. For things that are bools, can you convert true values to the string value "t" and false values to the string value "f"? For numbers, can you make them a string with quotes around them?
	- Here is the CoPilot query to generate the DataResults:
	> Can you build an array of DataResults where each DataResult is populated by the values in create_database.cpp and the metadata for the column names comes from the files in db_schema. Can you just do the classes and and people tables. Also, can you convert the array of DataResults to JSON before showing it to me? Note that the sortedColumnNames in DataResults should be sorted. Please sort the column names and adjust the data rows accordingly. Can you have the resulting JSON be an array of JSON objects where there is one field called "tableName" that has the name of the table and the other is "dataResults" that has the DataResult?
	- Here is the CoPilot query to create the mock ServerAccess
	> Can you create a service provided in the root named ServerAccessMock that implements ServerAccess? Can you create an injection token named SERVER_ACCESS_IMPLEMENTATION_TOKEN that is used to provide this class like ServerAccessNetwork? Can you create an interface called TableData that has a field called tableName with a string that is a table name and then another field called dataResults that is of type DataResults. Can you have a variable in ServerAccessMock that is of type array of TableData that is used to implement various methods. In particular, AddItem finds the entry in the array matching the table_name in AddItemBody and then adds the value to the dataTable in DataResults. If the primary key is not provided, will you find the lowest number not already in the array as a primary key (probably named id) and add this value for the primary key? Can you do the same for AddItemFetchPrimaryKey but return the generated primary key. For GetTableRows, can you find the right table and then return the corresponding DataResults? For GetRowsByColumn, can you find the corresponding table and then make a copy of the data results array, sort the entries by the columnName in ascending or descending order based on the value of the ascending parameter and then divide the results into blocks of size pageSize and return a DataResults of the pageNumber-th page? For UpdateItem, can you locate the table with table_name in UpdateItem that matches the corresponding DataResults and then update the given values based on the field values in the field value if the value in the entry has the given column_name with a value matching the given column_value. For DeleteItem, can you look through the DataResults for the matching tableName to find an entry with the given columnValue for the given columnName matching the item and then remove this entry from the array?
	- Here is the CoPilot query used to generate ColumnDataInfo from inside database_rest_helper.cpp
	> This file is generating JSON for IPC. In particular GenerateColumnMetadata generates JSON metadata describing a column. Can you use the code in this function to create a typescript export interface declaration named ColumnDataInfo?  The fields inside if(adminColumnDataInfo.GetAdminColumnDataInfo are all optional so they should have a question mark after the field name. The ones outside the if block are mandatory. All the fields are string except primary_key is a bool, unique is a bool, nullable is a bool, required is a bool, max_length is an integer, and rows is an integer.

- 8/13 What I'm working on
	- ColumnDataInfo is wrong and updated it (see previous paragraph)
	- The interchange comes as all text but it needs to be converted to the typed fields

- 8/14 What I'm working on
	- Used Copilot to convert an all string JSON to a typed ColumnDataInfo:
	> Can you look at ColumnDataInfo.ts? I am doing IPC using JSON objetcts where all the key value pairs are strings and the values might be null. Can you write function called ColumnDataInfoFromJSON that takes an unknown object and returns a ColumnDataInfo? For fields in the unknown object that are null, can you convert them to unknown in the result? Also, if you use the Record utility class in this conversion, can you make sure that you access the fields with [<string_key_name>] instead of record.fieldName that will be an error? Please note that for the bool values, the string will be "t" for true and "f" for false.
	- Used CoPilot to create ServerAccessProxy class with:
	> Can you create a service provided in root that is called ServerAccessProxy that implements ServerAccess the interface? Can you have it take a ServerAccess by the injection token SERVER_ACCESS_IMPLEMENTATION_TOKEN and expose itself via an injection token named SERVER_ACCESS_TOKEN and exposes it selt in a way like ServerAccessNetwork? Have it forward all of the implementation of its methods to the ServerAccess that it contains.
	- Used CoPilot to create unit tests for ServerAccess.mock.ts with:
	> Can you add a file ServerAccess.mock.spec.ts alongside this file. In that file, can you do unit tests for AddItem, AddItemFetchPrimaryKey, GetTableRows, GetRowsByColumn, UpdateItem, and DeleteItem. For AddItem, can you check the current items with GetTableRows, add an item, and then verfify that the row was added? Can you do the same for AddItemFetchPrimaryKey but also make sure that the next available primary key value is returned? For GetRowsByColumn, can you set a pageSize of two and then iterate through the assorted pages to make sure that the correct items are returned? For UpdateItem, can you modify one of the existing items and then verify with GetTableRows that the item was updated correctly. For DeleteItem, can you loop through deleting items via primary key and verify with GetTableRows that each item is deleted until everything is gone.
	- When I try to run ng test, all kinds of shit is broken
		- I fixed a lot of it but the big ones remaining are columnDataInfoId and defaultValue. Remove those.

- 8/15 What I'm working on
	- Get the remaining build errors fixed (columnDataInfoId and defaultValue)
		- Used CoPilot to update each component with:
		> Can you update this component to add an input named value of type string? In ngOnInit, can you have it set the value to value, if present, and then default_value otherwise? Can you add a section to ngOnChanges to look for changes to value like dataInfo and set the value of the text input like it does for default_value? Please make sure to do setValidators and updateValueAndValidity. Can you go through the test spec file and change any uses of defaultValue to default_value? In the corresponding html file, can you switch columnDataInfoId for column_name and switch any indexed references on dataInfo that use camel case to use snake case?
	- Adding unit tests for the new value property
		- Created branch client_4_testing
		- Created unit tests with this Copilot prompt:
		> We added the value input property to the component. Can we add unit tests for it so that we verify if the value is set, the value gets updated in the UI and if the value in the UI is set the corresponding event it emitted? Can you also verify that if both the value and default_value are set, just the value is shown?
		- What to work on next:
			- Fix the composite control to support values and make sure it still works with all of the changes
				- Update the tests based on values
			- Get the config stuff wired up to have dev with mock, dev with network, and prod
				- For the network cases, use the environment files to fetch attributes like server name and port
				- Replace the files for the ServerAccess provider based on configuration
			- Create a DatabaseSchema service that caches the database schema and sets timers
				- Create a mock version of this as well that maybe just uses the network mock of ServerAccess
			- Create the row control that aggregates the composite controls for a table row
			- Create a table control or use Lucas's
				- Have a way to choose the table and do pagination

- 8/18 What I'm working on
	- Updating the composite control to support value types
		- Made branch client_5_composite
		- Updated the ts file with this Copilot query:
			- Can you add an optional input parameter named value of type string?
		- Updated the html file with this Copilot query:
			- Can you route the input parameter value to the input parameter for each of the assorted subcomponents that currently take a dataInfo?
		- Updated the test speck file with this Copilot query:
			- Can you add tests for each of the various subcomponents to test two things: the first is that if value is set, it gets routed to the component and gets displayed and the default value is not shown. The second is that value is not provided and each of the subcomponents shows the default value. Please provide two separate tests, for each case, for each subcomponent type.
		- Tests pass. Note that there was a UTC issue in the generated tested code for dates. Switched to comparing just localized dates and everything worked.
	- Making environment work
		- Under environments we have:
			- environment.ts - default / prod
			- environment.prod.ts - same as above
			- environment.development.ts - development oriented
		- What is in these files
			- production as a boolean value
			- A role property just in production that is blank
			- Do we need a server and port property? No... uses proxy on development and the server itself will work on prod. Nothing for local
		- src/app/portal/services/network_abstraction
			- There is a SeverAccess.ts and ServerAccess.mock.ts
			- mock should be used for local workflows and the regular for development and prod
		- angular.json
			- under build there is a configurations section that has entries for production and development. There is a fileReplacements section for development that replaces src/environments/environment.ts with the production version
			- under serve there is a configurations section that has production and development that specifies the build target
		- src/proxy.conf.json
			- Maps /api to 127.0.0.1:8080 non HTTPS
			- Won't be used on production (I think)
		- What I need to do:
			- In Angular.json, add a section in build and serve for local
				- To this, add fileReplacements for ServerAccess.ts just for local that replaces it with the mock version
				- Make the rest of the local sections otherwise look like development
			- Created branch client_6_proxy
			- Should be able to run with ng serve -c local
			- Done

- 8/19 What I'm working on
	- Create a DatabaseSchema service that caches the database schema and sets timers
		- Here is the Copilot prompt
		- Create a service called DatabaseSchemaService provided in root that takes in its constructor a ServerAccess injected via the token SERVER_ACCESS_TOKEN. Please have a member variable of type DatabaseSchema and an observable of DatabaseSchema. On class startup, please have it call ServerAccess::GetDBSchema. Please set a timer to call this again after a configurable timeout as well as if the previous observable has an error. Please keep setting timers to try again until it is successful. Once we successfully get a DatabaseSchema, please cache it and cancel the timers / observables. Please add a method calld GetDBSchema that returns the cached copy, if present, or makes another call on ServerAccess::GetDBSchema and then block until this call or one of the other calls completes.
		- Had to fix a mess up in angular.json that was keeping ng test from running
		- Created branch client_7_db_schema
	- Create the row control that aggregates the composite controls for a table row
		- Needs to have a table name
		- Takes an array of column names, an optional array of values, and then pulls the DatabaseSchemaService to send the optional value and ColumnDataInfo to all of the various controls that it makes on the fly. It also takes a bool that indicates update or create to determine if, on post, it does an update or a create. It should also have a back button to go back to where we came from. The control should also keep track of the value and column name of the primary key.
		- Needs to have a primary key name / value pair to do an update. 
		- Here is the Copilot text to create the component:
			- Under src/app/portal/components, please create a component called CompositeRowControlComponent. Please create the files composite-row-control.component.ts, composite-row-control.component.spec.ts, composite-row-control.component.scss, and composite-row-control.component.html. Please make it standalone. It needs the following input parameters: a string table name that is not optional, a string array of column names that is also not optional, a string primary key name that is optional, a string primary key value that is optional, and a bool that indicates if we are in create new or update mode. Have the constructor take a DatabaseSchemaService and ServerAccess that is injected via SERVER_ACCESS_TOKEN. In the HTML file, please have a new style (Angular 17+) ng for that creates a number of child CompositeControlComponents. Create one for each column name passed into the array. Use DatabaseSchemaService to lookup the ColumnDataInfo for each column and pass the ColumnDataInfo and the corresponding value for the option value array input parameter to each CompositeControlComponent. Have a button that will be called Create if we are in create mode or Update if we are not and have the event handler package up each output value from each child CompositeControlComponent and then package them up into an object and call either AddItem on ServerAccess for create mode or UpdateItem otherwise. For UpdateItem, please use the passed in primary key name / value for the column_name / column_value. 
		- The Copilot query worked pretty well. If wanted to use any three times so I had those migrated to Record. It also had an out put property I don't think I need.
		- I realized that I shouldn't be passing in the values as an input parameter for the update case. It should use the table name and primary key / value to lookup the row. I should add a server side operation though to do this. It should be GetRow(string tableName, string columnName, string value). Returns a DataResults but just a single row in the values. Then I should have the update case fetch this row instead of having it passed in.
		- I would like the following tested:
			- Please note that you can use ServerAccess.mock.ts to provide the ServerAccess for testing. It provides default data as well as a fake database metadata that can be used for testing.
			- Update use case where values are passed into the control and make sure all the corresponding 

- 8/20 What to work on next
	- Check in what I have now
		- Branch client_8_row_control
		- Submitted
	- Add the server side GetRow
		- client_9_server_get_row
		- Unit test CoPilot generation
		> Can you write tests GenerateGetRowSqlBasic / GetRowBasic that you place after GetRowsByColumnBasic? Can you use the comment in GenerateGetRowSql to help guide what you expect the SQL text to look like? Canyou model GetRowBasic after GetRowsByColumnBasic but make sure that just one row is returned?
		- 
	- Update the client for this as well including the mock for server access
	- Rev this code to use GetRow
	- Add the unit tests

- 8/22 What I'm working on
	- Getting this exception
	```
	Exception: AddRowToTable failed with: ERROR:  insert or update on table "admin_column_data_info" violates foreign key constraint "fk_admin_column_data_info_table_name"
DETAIL:  Key (table_name)=(classes) is not present in table "admin_top_level_tables".
	```
  - This fails for both admin_column_data_info and admin_column_friendly_names referencing admin_top_level_tables for both people and classed.
  - AdminColumnInfo is:
	  - column_data_info_id
	  - table_name - foriegn key ref to top level tables
	  - column_name
	  - label
	  - hint
	  - place_holder
	  - regex
	  - html_input_type
	  - required
	  - max_length
	  - default_value
	  - rows
  - What is getting passed in PopulateAdminColumnDataInfo
	  - table_name, column_name, label, hint, html_input_type, required
  - Similar in PopulateAdminColumnFriendlyNames
  - Just needed to add the top level tables and then sort the columns for test data 
  - Need to do rest helper and then add an endpoint and then modify the client stuff.
  > Using get_rows_by_column.h, get_rows_by_column.cpp, and get_rows_by_column_test.cpp as guides as well as src/sql_util/database_crud_helpers.h, src/sql_util/database_crud_helpers.cpp, and src/sql_util/database_crud_helpers_test.cpp, add get_row.h, get_row.cpp, and get_row_test.cpp to wrap DatabaseRESTHelper::GetRow. Please add similar tests to get_rows_by_column_test.cpp but using GetRowBasic also as a guide. Please use "/api/get_row/\<string>/\<string>/\<string>" as the URL.
  - Completed the server
  - What needs to be done now:
	  - Modify ServerAccess and especially the mock and tests to support an add_row
	  - Change the row composite control to use this to fetch the control instead of passing it in
	  - Keep working on the row control and get unit tests up and running
	  - Get back to making a table or getting Lucas's table to work
  - Working on modifying ServerAccess to support GetRow
	  - Created branch client_10_sever_access
	  - Copilot query to implement GetRow on the mock
	  > Can you add the GetRow method? It should look for the table and then walk the rows in that table until it finds one where the the value of the given column matches the value passed into the method. It returns a DataResults with the columns sorted alphabetically but the table part will only have the single row in it.
	  - Copilot query to add unit tests for GetRow
	  > Can you add unit tests for the GetRow method? In particular, can you populate the table with rows, and then search for a row that exists and make sure that the correct data is returned, that the columns are sorted alphabetically, that the data corresponding to each column is in sync and that only one row is returned? Can you do another test for a non existent table and then another for a row that does not exist and then another for a data value that does not exist.
	  - Finished implementation and submitted.
  - Working on modifying CompositeRowControlComponent to get rid of the optional array of values for the update case and call GetRow on ServerAccess instead
	  - Copilot query to make this change.
	  > Can you modify this class to get rid of the optional input parameter value and instead use ServerAccess.GetRow on the update (ie. non create case) to fetch values that are passed into each subcontrol's value input property in the HTML file? You will need to find each of the columnNames in the sortedColumns array of the DataResults to find the corresponding value to use. Please update this file and the HTML file.
	  - Got this implemented with the ts and html file and created the branch client_11_composite_row_get_row
  - Started work on the composite control class unit tests
	  - Created branch client_12_composite_row_tests
	  - Copilot query to do the unit tests
	  > Can you create unit tests for the CompositeRowControlComponent class? Please use ServerAccessMock for the ServerAccess passed to the constructor. Please use the table people for the tests. Please do a test in create mode where you fill in all the values in each of the nested components created by the for in the html and verify that simulating triggering the Create button causes a row to get added to the ServerAccessMock. Please do another test in update mode for the given primary key and value that exists in ServerAccessMock and make sure that the values appear up in the client controls created by the html file by the for. Then update the edit control in each of these nested child components and simulate triggering the Update buttona and verify that row is modified in ServerAcccessMock.
	  - Added the tests and got them running. Submitted branch.
  - Wow! Need to look at Lucas's work and try to get his table up and running. Either convert his to use my metadata or create my own. I'd rather use his though...

8/24 Integrate Lucas's edit db table
- src/app/auth/services/db-crud.types.ts
- Table
```typescript
export interface DbTable {
  name: string;
  displayName: string;
  primaryKey: string;
  columns: DbTableColumn[];
}
```
- Column
```typescript
export interface DbTableColumn {
  key: string;
  dbType: DbColumnType;

  displayName?: string;
  editFieldType?: DbColumnEditField;
  editable?: boolean;
  defaultValue?: any;

  optionStrings?: string[];
  optionNumbers?: number[];

  uiFields?: any;
}
```
- Properties
```typescript
export type DbColumnType = 'UNIQUEIDENTIFIER' | 'VARCHAR' | 'INT';
export type DbColumnEditField =
  | 'text'
  | 'longtext'
  | 'email'
  | 'number'
  | 'optionstring'
  | 'optionnumber';
```
- New metadata
- Table
```typescript
export interface TableSchema {
  columns: ColumnDataInfo[];
  description: string; // Friendly description of the table
  foreign_keys: ForeignKeyInfo[];
  primary_key: string; // Column name of the primary key
  table_friendly_name: string; // Friendly name of the table, e.g., "People
  table_name: string; // Table name in the database, e.g., "people"
}
```
- Table differences
	- name -> table_name
	- displayName -> table_friendly_name
	- primaryKey -> primary_key
	- columns -> columns
- Column
```typescript
export interface ColumnDataInfo {
    column_name: string;
    type: string;
    primary_key: boolean;
    unique: boolean;
    nullable: boolean;

    column_friendly_name?: string;
    label?: string;
    hint?: string;
    place_holder?: string;
    regex?: string;
    html_input_type?: string;
    required?: boolean;
    max_length?: number;
    default_value?: string;
    rows?: number;
}
```
- Column differences
	- key -> column_name
	- dbType -> type
	- displayName -> column_friendly_name
	- editFieldType -> html_input_type
	- editable -> not sure that this is a concept we need
	- defaultValue -> default_value
	- optionStrings -> not sure
	- optionNumbers -> not sure
	- uiFields -> not sure
	- Need to map unique, nullable, label, hint, place_holder, regex, required, max_length, and rows
- What to do next
	- Get app up and running
	- Migrate the old names to the new names
	- Migrate the old types to the new types
	- Migrate from the old services to the new services
	- Migrate to wire up the create / edit / delete support

8/25 What I'm working on
- Have the app up and running. Click on login and then go to Admin and then Manage Users
- Trying to understand how all of this works:
	- src/app/auth/components/table-entry-form-dialog
		- Angular lets you pass JSON when you call open on a dialog and you access this data from the dialog by using the injection token MAT_DIALOG_DATA
		- This brings up the little dialog with the ability to edit things with a Save / Cancel button
		- I might want this to bring up another page though as we do nested items in SQL and I definitely want to use my controls for this instead of just all edit boxes.
		- But I can go to where this is opened to understand the flow better.
		- This is used in edit-db-table.component.ts in addRowToTable and clickEdit
	- src/app/auth/components/admin
		- This takes in the DbCrudService, Router, and ActivatedRoute through the constructor
		- This sets selectedTableName on DbCrudService based on the ActivatedRoute param tableName
		- Based on the selectedTableName, we call Router.navigate
		- HTML file has a router-outlet tag
	- src/app/auth/components/edit-db-table
		- src/app/auth/auth.routes.ts
			- This is the routing for admin and there is a children section under routing with path: ':tableName' and this component
		- Constructor takes a DbCrudService and a MatDialog
		- It identifies the primary key column name
		- This component can getSelectedTableData, sortByColumn, addRowToTable, clickEdit, clickDelete
	- src/app/auth/services/db-crud.service.ts
		- Has these properties:
			- dbTables$(get): Observable\<DbTable\[\]>
			- selectedTableName$(get, set): Observable\<string>
			- selectedTable$(get): Observable\<DbTable>
		- This is the place stubbed out to make network calls
- Rename fields to prep for type switch over
	- Branch client_13_rename
	- src/app/auth/services/db-crud.types.ts
	- Here is the Copilot query to do the rename for DbTableColumn
	> I would like to rename fields in DbTableColumn. Please rename the fields in this file as well as all the users of these fields in other files. Here is the list of field name changes: key -> column_name, dbType -> type, displayName -> column_friendly_name, editFieldType -> html_input_type, defaultValue -> default_value. Please only do renames for DbTableColumn and not DbTable.
	- Here is the Copilot query to do the rename for DbTable
	> I would like to rename fields in DbTable. Please rename the fields in this file as well as the users of these fields in other files. In particular, you have struggled with finding usages in the html files. Please only do this for DbTable. Here is the list of field name changes: name -> table_name, displayName -> table_friendly_name, primaryKey -> primary_key.
	- Submitted branch
- Handle fields that no longer exist
	- In DbTableColumn, these fields don't map to anything anymore:
		- optionStrings
			- Not used
		- optionNumbers
			- Not used
		- uiFields
			- There is a sortDirection on the uiFields that nothing is currently setting
			- In sortByColumn, we use this to make a decision on which direction to sort and then toggle it for the next time. Just add a sortDirection here to accomplish the same thing and remove the field.
		- Migrated to a Record to accomplish the same thing and verified that it works
	- Removed the fields and verified tests still pass and page still works
	- Created branch client_14_remove_unused
	- Submitted
- Want to move over to the new types. DbTable / DbTableColumn are pretty much ready to migrate. Now we need to look at DbRow and DbEntry
	- DbRow isn't used anywhere oddly enough
	- DbEntry isn't used either
	- What uses DbTable:
		- admin.component.ts
			- Has an array of DbTable called dbTables
			- Does a subscribe on DbCrudService.dbTables$
		- edit-db-table.component.ts
			- selectedDbTable: DbTable
			- subscribes to DbCrudService.selectedTable$ to set selectedDbTable
			- tableDataValues is of type any and subscribes to DbCrudService.getTableValues
				- What type is this?
			- How does DbCrudService.compareItems work?
			- addRowToTable needs to fetch the data values from the dialog and push them to the service
			- clickEdit needs to fetch the data values from the dialog and push them to the service
			- clickDelete needs to make the call to the service to remove the values
		- table-entry-form-dialog.component.ts
			- calls updateItem on DbCrudService
		- db-crud.service.ts
			- get dbTables$: Observable\<DbTable[]> should be easy to map to new service
			- deleteItem / updateItem / getTableValues / refreshDbTables / getTableValues - map to new service
		- mock-db-crud-response.ts
			- This file should just go away eventually but can easily map to our new types
	- The plan:
		- Rename the types to be the same as the new types
		- Migrate to DbCrudService being a shim
		- Remove DbCrudService and replace with direct calls to ServiceAccess

8/25 What I'm working on
- Rename the types to be the same as the new metadata
	- Created branch client_15_type_rename
	- Removed DbRow and DbEntry
	- Rename DbTable to TableSchema
		- Copilot query for this
		> Can you rename DbTable to TableSchema? Please do it in this file and all places that include this file.
	- Rename DbTableColumn to ColumnDataInfo
		- Copilot query for this
		> Can you rename DbTableColumn to ColumnDataInfo? Please do it in this file and the other places that import this file like: admin.component.ts, edit-db-table.component.ts, table-entry-form-dialog.component.ts, db-crud.service.ts, and mock-db-crud-response.ts
- Migrate to use the server types and get rid of the client types
	- Created branch client_16_server_types
	- Copilot query
		> Switch all usages of this file that use TableSchema to TableSchema from @shared/types/TableSchema.ts and all usages of this file that use ColumnDataInfo to ColumnDataInfo in @shared/types/ColumnDataInfo.ts. There should be no usages of this file left in the project. Please look in: admin.component.ts, edit-db-table.component.ts, table-entry-form-dialog.component.ts, db-crud.service.ts, and mock-db-crud-response.ts
- Migrate to DbCrudService being a shim
	- Plan
		- Inject a ServerAccess with SERVER_ACCESS_TOKEN
		- Implement dbTables$(get) by subscribing to ServerAccess.GetDBSchema() and then fetching the tables out of DatabaseSchema
		- Implement deleteItem with ServerAccess.DeleteItem()
		- Implement updateItem with ServerAccess.UpdateItem()
		- Implement \_getDbTablesResponse by subscribing to ServerAccess.GetDBSchema() and then fetching the tables out of DatabaseSchema
		- Implement \_getDbTableValuesResponse with ServerAccess.GetTableRows/GetRowsByColumn
	- Created branch client_17_crud_service
	- Copilot query to inject SeverAccess
	> Can you add a constructor parameter that injects a ServerAccess using SERVER_ACCESS_TOKEN?
	- Copilot prompt to create a function like dbTables$(get) but using the service
	> I'd like to add a public property with just a get function called myDbTables$ of type Observable<TableSchema[]>.  Please implement this by subscripting to ServerAccess.GetDBSchema and returning DatabaseSchema.tables in an observable.
	- I have migrated to using ServerAccess, dbTables$, deleteItem, and updateItem
	- \_getDbTableValuesResponse() is hard because it is assuming that the values are in the same order as the columns. The issue is that the values are in a DataResults and are sorted but the columns aren't. The correct solution is to sort the columns AND convert things to use DataResults but I will sort the values based on the columns for now.
	- Copilot query to do this:
	> GetTableRows returns and Observable\<DataResults>. This has a list of sortedColumnNames and dataTable that is an array of arrays of strings where each is an array of representing rows that are each an array of column values corresponding to the same index as the column name in sortedColumnNames. I need to return an Observable that is an array of objects that have column name / column value key/ value pairs. Can you write a function myGetTableValuesResponse that does this and takes a table name as a string and returns Observable\<any> and does this behavior?
	- This works for local now!
	- What to work on tomorrow:
		- Get networking working and check it in
		- Figure out if it used to automatically show a table when you opened the portal
		- Switch to using DataResults natively
		- Switch to using DatabaseSchema and top level tables
		- Switch over to using ServerAccess natively
		- Either use the DatabaseSchema service or get rid of it
		- Add pagination support
		- Have the edit / insert stuff be a new page instead of a dialog

8/28 What I'm working on
- Get networking working
	- According to proxy.conf.json the server is at: http://127.0.0.1:18080
	- It looks like a valid URL is: http://127.0.0.1:18080/api/get_db_schema
	- http://127.0.0.1:18080/api/get_table_rows/classes
	- Getting this error accessing the database:
	> ERROR:  database "knottyyoga" is being accessed by other users DETAIL:  There is 1 other session using the database.
- Try accessing from the command line
- If I use an absolute URL to the resource I get:
> Access to XMLHttpRequest at 'http://127.0.0.1:18080/api/get_db_schema' from origin 'http://localhost:4200' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
- You can add headers on the server side but it is probably better to do this on the client side. 
- On the client side, you need to start with:
> ng serve --proxy-config proxy.conf.json
- Taking out the absolute URLs and trying proxyconf
- Added "proxyConfig": "src/proxy.conf.json", to the serve section for development. It ALL WORKS!
- Sent out the merge request

8/29 What I'm working on
- Things to do
	- Switch to using DataResults natively
	- Switch to using DatabaseSchema and top level tables
	- Change the the AddItem/UpdateItem to take the same parameters as ServerAccess
	- Switch over to using ServerAccess natively
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog
- Switch to using DataResults natively
	- Created branch client_18_dataresults
	- Main thing is getTableValues
	- Only used in src/app/auth/components/edit-db-table/edit-db-table.component.ts
	- We subscribe in getSelectedTableData() and set tableDataValues: any[]
	- This gets set to an empty array and then called in ngOnInit
	- Inside sortByColumn, we call sort on the tableDataValues but this could be dataTable on DataResults
	- clickEdit calls to the dialog with a row but the current row is an object with name / value pairs whereas we most likely would need to pass a DataResults object with either an index or a new one with just that one row. The full thing with the index might be easier. It also currently returns an updated result which might be easier to just return the whole DataResults
	- clickDelete splices out the object
	- tableDataValues is used in edit-db-table.component.html but Mainly to provide to provide values to clickEdit/clickDelete (which could be indexes) and we look up the value by column name but we could use a helper that looks it up in sorted column names instead to find the index and do it that way.
	- TableEntryFormDialogComponent takes a TableSchema and value in as a parameter. I can easily and more typesafe pass in a DataResults and index here and then return the DataResults.
	- Plan
		- In DbCrudService
			- Change getTableValues() to be called GetTableRows() and return Observable\<DataResults>
			- Get rid of \_getDbTableValuesResponse
		- In EditDbTableComponent
			- Rename tableDataValues to dataResults and change the type to DataResults
			- In getSelectedTableData, rename tableDataValues, set the value, and switch to the right type
			- In ngOnInit, rename tableDataValues and set to an appropriate set of default values.
			- In sortByColumn, we need to rename tableDataValues and sort the dataTable
			- In clickEdit, change to take an index as a parameter and pass a DataResults and index and then fetch out the DataResults that gets returned. Might need to do the spread operator to create a new object to get dependency injection to fire. Most likely do this in tandem with TableEntryFormDialogComponent
			- In clickDelete, change to take an index in as a parameter and we need to splice out the object
			- In the HTML file, change the clickEdit/clickDelete to take an index
			- In the HTML file, rename tableDataValues and write a helper function to take a column name and return an index into dataTable
		- In TableEntryFormDialogComponent
			- Change TableEntryFormDialogData to take a DataResults in
			- Change TableEntryFormDialogResult to return a DataResults
			- In ngOnInit, lookup the sortedColumnName / index to fetch the value
			- In save, switch to using the DataResults and returning the right value
	- Implementation
		- Created branch client_18_dataresults
		- In DbCrudService, made the changes but commented out the old code with a todo
		- In EditDbTableComponent:
			- Commenting stuff out with TODOs
			- Replaced tableDataValues with dataResults
			- In ngOnInit, set dataResults to empty
			- In getSelectedTableData, switched to subscribing to GetTableRows
			- In sortByColumn, use this Copilot query to rewrite the sort:
			> in sortByColumn, we previously had an array tableDataValues that is an array of rows and we would sort the rows by the indicated column. The data was an object that was essentially a set of key / value pairs. This meant that each object had its's own copy of all of the keys / column names. Now I am moving to DataResults that has two arrays. One is a set of sortedColumns that has the columnNames once. The other is an array of arrays of strings where each item in the top level array is a set of rows with each secondary array just being a specific column value so the keys are normalized. Each value in sortedColumns corresponds by index to the column in each row in dataTable. Can you rewrite this sort function to find the index of the column in sortedColumns and then use this index to do the comparison of the equivalent column by index in each a / b row in dataTable?
			- In clickEdit, value is currently a row object with key/value pairs. Changed this to a DataResults and index. Assign dataResults from the return.
				- Need to update the HTML and the dialog code
				- Updated the return to fetch the single row out of DataResults and update in place
			- In clickDelete, switched to index and did the splicing
			- In HTML file, switched clickEdit and clickDelete to be by index. Switched to iterating over dataResults.dataTable
			- In addRowToTable, comment out the code with a TODO since it isn't used
		- In TableEntryFormDialogComponent
			- Change  TableEntryFormDialogData and TableEntryFormDialogResult to DataResults. Note that the result is just a single row
			- Change ngOnInit to use the DataResults
			- In save, generate the old school object to pass to updateItem and build a single row DataResults to return
		- I think that everything is done but I haven't built or tested

9/3 What I'm working on
- Getting the DataResults stuff working
	- Builds
	- ng test runs
	- ng serve works locally and via the network
	- Remove the comments and check it in
- Things to do
	- Switch to using DatabaseSchema and top level tables
	- Things are getting fucked up on update
	- Switch to using DestroyRef
	- Change the the AddItem/UpdateItem to take the same parameters as ServerAccess
	- Switch over to using ServerAccess natively
	- Add support for adding rows
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog
- Switch to using DatabaseSchema and top level tables
	- Created branch client_19_database_schema
	- The plan
		- How does Subscription work in Angular?
			- The type that gets returned from Observable\<>.subscribe
			- Can call add()  on a subscription to register a function that will get called on unsubscribe from the subscription
		- How does BehaviorSubject work in Angular?
			- A stateful stream that always has a value and replays that value immediately to new subscribers
			- Supports:
				- Pushing the next value with next(value)
				- Synchronous read with .value or .getValue()
				- Expose as a readonly observable .asObservable()
		- Craft a Copilot query to better understand the code:
			> I'm trying to remove _dbTables in DbCrudService. It's a BehaviorSubject initially containing undefined that is populated with unknown. The only place that it is used is calling next from within refreshDbTables(). I don't see how this accomplishes anything since it's not used anywhere else. We return a Subscription from refreshDbTables(). Can you help me understand what the call to add on the Subscription is doing inside loadDbTables(). Is there a way to accomplish this same thing without a subscription?_
			- 
		- DbCrudService
			- Rename dbSchema to _databaseSchema
			- Rename dbTables$ to databaseSchema$ and convert the type
			- Get rid of \_dbTables and figure out what BehaviorSubject is
			- Rewrite selectedTable$ to hack into databaseSchema
			- Get rid of \_getDbTablesResponse and convert refreshTables to speak to the DatabaseSchema (actually just get rid of these)
		- EditDbTableComponent
			- Nothing to change
		- TableEntryFormComponent
			- Nothing to change
		- AdminComponent
			- Change dbTables to DatabaseSchema
			- ngOnInit to subscribe to databaseSchema$
			```typescript
import { Component, DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class MyComponent {
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.form.valueChanges
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(v => this.saveDraft(v));
  }
}			
			```
			- Note that we should migrate to using DestroyRef
			- Just do the equivalent of loadDbTables inside ngOnInit
			- Write a getFriendlyName(tableName) that takes a tableName, finds the table, and then fetches the friendly name
			- Convert the HTML template to use topLevelTables and getFriendlyName
	- Implementation
		- DbCrudService
			- Removed \_dbTables
			- Convert dbTables$ to databaseSchema$
			- Rewrote selectedTable$ to use DatabaseSchema
			- Get rid of \_getDbTablesResponse and refreshDbTables
		- AdminComponent
			- Change dbTables to databaseSchema
			- In ngOnInit, subscribe to databaseSchema$
			- Doing the equivalent of loadDbTables inside ngOnInit
			- Wrote getFriendlyName
			- Switched over the HTML template
	- It all seems to work with ng test and running with ng serve. Check it in :)
- Values after update get out of order
	- Looking at TableEntryFormDialog.save
		- There is a TableSchema called tableData and we enumerate columns and then sort them to generate sortedColumnNames
			- We don't add the primary key
		- We create a value that is a Record\<string, string> to store the updated value
		- We walk the form controls for name / control
			- Found the bug I think. We generate the the columnIndex based on the input dataResults.sortedColumnNames instead of the one here.
	- Created branch client_20_update_issue
	- Verified that this fixes the issue
- Switch over to destroyRef
	- Created branch client_21_destroy
-  Things to do
	- Change the the AddItem/UpdateItem to take the same parameters as ServerAccess
	- Change ServerAccess void methods to be observable of void
	- Move the selectedTableName out of DbCrudService
	- Move compareItems out of DbCrudService
	- Change deleteItem to match ServerAccess
	- Switch over to using ServerAccess natively
	- Add support for adding rows
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog
```typescript
export interface AddItemBody {
  table_name: string;
  value: unknown; // Object with key value pairs to insert into database
}

export interface UpdateItemBody {
  table_name: string;
  value: unknown; // Object with key value pairs to insert into database
  column_name: string; // Name of column to update by
  column_value: string; // Name of column
}

export interface ServerAccess {
  AddItem(body: AddItemBody): void;
  AddItemFetchPrimaryKey(body: AddItemBody): Observable<string>;
  UpdateItem(body: UpdateItemBody): void;
}
```
-  Change the the AddItem/UpdateItem to take the same parameters as ServerAccess
	- UpdateItem 
		- This is currently updateItem(typeName: string, item: any) : Observable\<any>
			- typeName is the tableName
			- item is a key / value pair item that we have to fish the primary key out of
		- The primary key generally does not change and should be separated.
		- The only caller is TableEntryFormDialogComponent
	- AddItem
		- There currently is no AddItem support so it can't get any worse
	- Plan
		- DbCrudService
			- Add UpdateItemBody as a public export
			- Convert method to be a pass through to ServerAccess
		- TableEntryFormDialogComponent
			- Modify save method to build the UpdateItemBody and call the new method
	- Implementation
		- Created branch client_22_update
		- DbCrudService
			- Creating UpdateItem but with a body of observable of void
		- TableEntryFormDialogComponent

9/4 What I'm working on
- Things to do
	- Change ServerAccess void methods to be observable of void
	- Change deleteItem to match ServerAccess
	- Move the selectedTableName out of DbCrudService
	- Remove the unused imports from DbCrudService
	- Get rid of databaseSchema$
	- Move compareItems out of DbCrudService
	- Rename ServerAccess methods to lower case
	- Switch over to using ServerAccess natively
	- Add support for adding rows
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog
- Change ServerAccess void methods to be observable of void
	- The plan
		- Methods that currently return void: AddItem, UpdateItem, DeleteItem
		- Mostly rename the methods and users but also update the mock tests
	- Implementation
		- Created branch client_23_fire_and_forget
		- Did the type change in all the ServerAccess files
		- In src/app/portal/services/network_abstraction/ServerAccess.mock.spec.ts I am now using the observable
		- DbCrudService
			- Mapped deleteItem from Observable void to bool
			- Just pass through UpdateItem
- Change deleteItem to match ServerAccess
	- The plan
		- DbCrudService
			- The method currently just takes the value of the primary key
			- Need to change this to take a key and value and defer that to the client
		- EditDbTableComponent
			- We already have the primary key name at the only caller
	- Implementation
		- Created branch client_24_delete_item
			- DbCrudService
- Move the selectedTableName out of DbCrudService
	- The plan
		- We have public get/set for selectedTableName$
			- Only used by AdminComponent
			- We assign it in ngOnInit (twice)
			- In ngOnInit, we also subscribe to the property as an Observable
		- We have a public getter for selectedTable$ that returns an observable of a TableSchema
			- This is a strange combineLatest that subscribes to both the selectedTableName$ and databaseSchema$ and uses the two to produce an observable of TableSchema.
			- This is only used by EditDbTableComponent
			- Unfortunately, it might be better to create a more appropriately named TableManagementService and move this logic there and out of DbCrudService
	- Implementation
		- Created branch client_25_table_management
		- Here is the Copilot prompt to create the service
		> I would like to factor some logic out of DbCrudService and move it into another service, also provided in root, in the same directory named TableManagementService that lives in a file table-management.service.ts. Please create a test file named table-management.service.spec.ts. For now, just create the service and don't remove the code from DbCrudService or change any of the users. I'd like to have a constructor that injects ServerAccess like DbCrudService. Please copy over _selectedTableName and selectedTableName$/selectedTableName directly. I don't want databaseSchema$ but do want selectedTable$ but replace the databaseSchema$ in combineLatest with a call to ServerAccess.GetDBSchema() directly.
		- Created TableManagementService

9/5 What I'm working on
- Things to do
	- Get rid of databaseSchema$
	- Move compareItems out of DbCrudService
	- Rename ServerAccess methods to lower case
	- Switch over to using ServerAccess natively
	- Bug: sort users by email and then delete kit. Two users go away.
	- Add support for adding rows
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog
- Get rid of databaseSchema$
	- Only used in AdminComponent, just use method on ServerAccess
	- Created branch client_26_remove_db_schema
- Move compareItems out of DbCrudService
	- Only used in EditDbTableComponent, move it to there
	- Created branch client_27_compare_items
- Rename ServerAccess methods to lower case
	- Created branch client_28_lower_case
	- Copilot query
	> I'd like to rename all of the ServerAccess methods to lowercase. Please do this in all the classes that implement this interface as well as all of their users. Also, please rename the methods of DbCrudService to lowercase as well and all of their call sites.
	- Worked okay for ServerAccess but not for DbCrudService, new query:
	> Please rename all of the methods of DbCrudService to lower case. Please go and rename all of the usages of these methods at the call sites as well.
- Switch over to using ServerAccess natively
	- Created branch client_29_remove_db_crud_service
	- Used in EditDbTableComponent and TableEntryFormDialogComponent
- Bug: sort users by email and then delete kit. Two users go away.
	- Created branch client_30_delete_bug
	- The core issue is that the db schema returned by getDbSchema is by reference so the splice in edit db table is double deleting. Switch to a deep copy

9/6 What I'm working on
- Things to do
	- Add support for adding rows
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog

9/10 What I'm working on
- Add support for adding rows
- The plan
	- Currently shows New value in {table}
	- There is a EditDbTableComponent.addRowToTable already
		- It opens the dialog and the primary key is shown as a field
		- It doesn't end up doing anything
		- Need to hide the primary key when sending the DataResults to the dialog and then call AddRowAndFetchPrimaryKey then add to result set
			- Create a blank data results to pass in to child dialog
			- Set valueIndex to -1
			- Pass in tableData
			- Make the call and then splice out the lone result value
	- TableEntryFormDialogComponent
		- valueIndex -1 means that we are doing add item
		- We should remove the primary key from all cases
		- Need to call the add and fetch primary key and return that
- Implementation
	- Created branch client_31_add_row
	- TableEntryFormDialogComponent
		- Changed ngOnInit to set the default value for new item cases and set the required validator for required fields
		- Remove the primary key from the grid
		- Added update and addNew methods that save calls
			- Verified that update works
	- EditDbTableComponent
		- Pass -1 for value index for add new
	- ServerAccess
		- Return from add fetch primary key is JSON like {id: 4} so parse the value out and return that from the method
	- Completed
- Things to do
	- Either use the DatabaseSchema service or get rid of it
	- Add pagination support
	- Have the edit / insert stuff be a new page instead of a dialog
	- Add image support
	- Add parent / child support
# Branches
- client_1_column_data_info
	- Out for review
	- Submitted
- client_2_interfaces
	- Out for review
	- Submitted
- client_3_network
	- Out for review
	- Submitted
- client_4_testing
	- Out for review
	- Submitted
- client_5_composite
	- Out for review
	- Submitted
- client_6_proxy
	- Out for review
	- Submitted
- client_7_db_schema
	- Submitted
- client_8_row_control
	- Submitted
- client_9_server_get_row
	- Submitted
- client_10_sever_access
	- Submitted
- client_11_composite_row_get_row
	- Submitted
- client_12_composite_row_tests
	- Submitted
- client_13_rename
	- Submitted
- client_14_remove_unused
	- Submitted
- client_15_type_rename
	- Submitted
- client_16_server_types
	- Submitted
- client_17_crud_service
	- Submitted
- client_18_dataresults
	- Submitted
- client_19_database_schema
	- Submitted
- client_20_update_issue
	- Submitted
- client_21_destroy
	- Submitted
- client_22_update
	- Submitted
- client_23_fire_and_forget
	- Submitted
- client_24_delete_item
	- Submitted
- client_25_table_management
	- Submitted
- client_26_remove_db_schema
	- Submitted
- client_27_compare_items
	- Submitted
- client_28_lower_case
	- Submitted
- client_29_remove_db_crud_service
	- Submitted
- client_30_delete_bug
	- Submitted
- client_31_add_row
	- Submitted
- 
# Tasks
- List of tasks here with status