中文版本请[点击这里](https://github.com/cmwsssss/CCDB/blob/main/README-CN.md)

# What's CCDB
CCDB is a high-performance database framework based on Sqlite3 and Swift, ideal for SwiftUI development

CCDB has an OBJC version , OBJC version is faster , support for dictionary->model mapping , Less code required to use, developers who use OBJC [click here](https://github.com/cmwsssss/CCDB-OBJC)

## Features

#### Easy-to-use:
CCDB is very easy to use, Just one line of code to insert, query, delete and update, The programmer does not need to be concerned with any underlying database level operations, such as transactions, database connection pooling, thread safety, etc, CCDB will optimize the API operations at the application level to ensure efficient operation at the database level

#### Efficient:
CCDB is based on the multi-threaded model of sqlite3 and has a separate memory caching mechanism, making its performance better than direct use sqlite3 in most cases.

* Performance comparison with Realm (based on the same data model):
    <img width="960" alt="截屏2021-12-13 下午2 41 51" src="https://user-images.githubusercontent.com/16182417/145764680-a771955d-7cd2-4db3-9df9-0e912553572b.png">

**In terms of write speed, CCDB is faster than Realm, but in terms of query, CCDB is weaker than Realm**
    
* CCDB provides memory cache, which will greatly increase the speed when data needs to be queried twice or more.
    <img width="960" alt="截屏2021-12-13 下午2 43 23" src="https://user-images.githubusercontent.com/16182417/145764726-99bd59e2-35eb-4f40-8602-f1179d3d4091.png">  


#### SwiftUI adaptation::
CCDB has optimized the SwiftUI adaptation, and the model properties are adapted to the @Published, meaning that any change in the value of properties will cause the UI to be refreshed

#### Container:
CCDB also provides a list solution: **Container**, which makes it very easy to save and read list data.

#### Singleton:
The object generated by CCDB will only have one copy in memory, which is the basis for adapting SwiftUI

## Getting Started

#### Prerequisites
Apps using CCDB can target: iOS 13 or later.

#### Installation
pod 'CCDB'

#### Initialize
Call the initialization method before using the CCDB's API
```
CCDBConnection.initializeDBWithVersion("1.0")
```
If the data model properties have changed and you need to migrate the database, just change the verson

#### Object Relational Mapping(ORM)

##### Inheritance of CCModelSavingable protocol
**Note: CCDB models must have a primary property, which is the first property of the model properties**
```
class UserModel: CCModelSavingable {
    var userId = "" //primary property
    ...
}
```
##### Implement the modelConfiguration method 
```
static func modelConfiguration() -> CCModelConfiguration {
    var configuration = CCModelConfiguration(modelInit: UserModel.init)
    ...
    return configuration
}
```
CCDB supports the following types: Int, String, Double, Float, Bool and classes that inherit from CCModelSavingable.

##### 3. Custom Types：
If there are some types in the model properties that CCDB does not support, such as array, dictionary, or non-CCModelSavingable objects, some additional code is needed to codec these data and then save and read them.
```
class UserModel: CCModelSavingable {
    var userId = "" //primary property
    var photoIds = [String]()  //Array
    var height: OptionModel?  //non-CCModelSavingable class
}
```

```
//Configure the special properties
static func modelConfiguration() -> CCModelConfiguration {
    var configuration = CCModelConfiguration(modelInit: UserModel.init)
    //The photoIds is a custom type and needs to be handled manually
    configuration.inOutPropertiesMapper["photoIds"] = true  
    
    //The height is a custom type and needs to be handled manually
    configuration.inOutPropertiesMapper["height"] = true  
    
    //Encoding Method
    configuration.intoDBMapper = intoDBMapper 
    
    //decoding Method
    configuration.outDBMapper = outDBMapper
    ...
    return configuration
}
```

* encode custom data as JSON strings
```
static func intoDBMapper(instance: Any)->String {
        
    guard let model = instance as? UserModel else {
        return ""
    }
        
    var dicJson = [String: Any]()
    dicJson["photoIds"] = photoIds
        
    if let height = model.height {
        dicJson["height"] = height.optionId
    }
        
    do {
        let json = try JSONSerialization.data(withJSONObject: dicJson, options: .fragmentsAllowed)
        return String(data: json, encoding: .utf8) ?? ""
    } catch  {
        return ""
    }
}
```
* Decode and populate properties with JSON strings from the database

```
static func outDBMapper(instance: Any, rawData: String) {
    do {
        guard let model = instance as? UserModel else {
            return
        }
        if let data = rawData.data(using: .utf8) {
            if let jsonDic = try JSONSerialization.jsonObject(with: data, options: .fragmentsAllowed) as? Dictionary<String, Any> {
                if let photoIds = jsonDic["photoIds"] as? [String] {
                    model.photoIds = photoIds
                        }
                    }
                }
                    
                if let heightId = jsonDic["height"] as? String {
                    model.height = OptionModel(optionId: heightId)
                }
            }
        }
    } catch  {
            
    }
}
```
#### Support @Published：
If you want the model property values to be bound to SwiftUI page elements, you need to wrap the properties using @Published, and these wrapped properties also need to be configured within the modelConfiguration.

```
class UserModel: CCModelSavingable {
    var userId = "" //primary property
    @Published var username = ""
    @Published var age = 0
    ...
}
```
* The type of the property needs to be put into the Mapper, and the value of the key is **_propertyName**
```
static func modelConfiguration() -> CCModelConfiguration {
    var configuration = CCModelConfiguration(modelInit: UserModel.init)
    //_username is the key, value is the type of the username
    configuration.publishedTypeMapper["_username"] = String.self   
    configuration.publishedTypeMapper["_age"] = Int.self
    ...
    return configuration
}
```

#### Update and insert
For CCDB, the operations are based on CCModelSavingable objects, **objects must have primary property**, so update and insert are the following code, if there is no data corresponding to that primary key within the data, it will be inserted, otherwise it will be updated.

**CCDB does not provide batch write, CCDB will automatically create write transactions and optimize them**
```
userModel.replaceIntoDB()
```

#### Query
CCDB support query by primary key, batch queries and conditional queries

##### Query by primary key
Get the corresponding model object by primary key
```
let user = UserModel.initWithPrimaryPropertyValue("userId")
```
##### Batch queries
* Get the count of the table
```
let count = UserModel.count()
```
* Get all objects of the model
```
let users = UserModel.queryAll(isAsc: false)
```

##### Conditional queries
The configuration of the CCDB condition is done through the object of the CCDBCondition
Query the first 30 users whose age is greater than 20 in the UserModel table, and return the results in reverse order by age
```
let condition = CCDBCondition()
//ccmethods are not sequential
condition.ccWhere(whereSql: "age > 30").ccOrderBy(orderBy: "age").ccLimit(limit: 30).ccOffset(offset: 0).ccIsAsc(isAsc: false)

//Query users according to the conditions
let res = UserModel.query(condition)

//Get count of users according to the conditions
let count = UserModel.count(condition)
```

#### Delete
* Delete an object
```
userModel.removeFromDB()
```
* Delete all objects
```
UserModel.removeAll()
```

#### Index
* Create index
```
//Create index for age
UserModel.createIndex("age")
```
* Remove index
```
//Remove index for age
UserModel.removeIndex("age")
```

#### Container
Container is a solution for list data, the value of each list can be written to Container, Container's table data is not a separate copy, its associated with the data table data

```
let glc = Car()
glc.name = "GLC 300"
glc.brand = "Benz"
// Assuming the containerId of the Benz car is 1, here the glc will be written into the list container of the Benz car
glc.replaceIntoDB(containerId: 1, top: false)

//Get the list data of all Benz cars
let allBenzCar = Car.queryAll(false, withContainerId: 1)

//Remove glc from the list of Benz cars
glc.removeFromDB(containerId: 1)
```
Container data access has also been optimized within CCDB

#### Adapted to SwiftUI
CCDB supports @Published, just add a few lines of code to notify the UI for updates when the property changes

```
class UserModel: CCModelSavingable, ObservableObject, Identifiable {
    var userId = ""
    @Published var username = ""
    ...
    
    //Implement the protocol method like this
    func notiViewUpdate() {
        self.objectWillChange.send()
    }
}

class SomeViewModel: ObservableObject {
    @Published var users = [UserModel]()
    init() {
        weak var weakSelf = self
        //Add this code to notify UI changes when UserModel properties are changed
        UserModel.addViewNotifier {
            weakSelf?.objectWillChange.send()
        }
    }
}

class SomeView: View {
    @ObservedObject var viewModel: SomeViewModel
    var body: some View {
        List(self.viewModel.users) {user in
            Text(user.username)
        }
    }
}
```
