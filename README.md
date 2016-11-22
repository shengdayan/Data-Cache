# Data-Cache
说到数据同步，首先涉及到数据缓存，在实际项目开发中常用的是MVC模式，即controller去控制model在view上的显示，有些项目甚至用到十几甚至几十个表结构，为了减少从后台请求加载，影响用户体验，很多数据可以缓存在本地从本地取。这里介绍一个好用的第三方LKDBHelper。

##LKDBHelper
写一个baseModel继承NSObject，在.h文件申明属性，定义方法：
```
- (instancetype)initWithDict:(NSDictionary *)dict;
+ (LKDBHelper *)getUsingLKDBHelper;
```  
在.m文件
```
- (instancetype)initWithDict:(NSDictionary *)dict {
    if (self = [super init]) {
        
    }
    return self;
}

//重载选择 使用的LKDBHelper
+(LKDBHelper *)getUsingLKDBHelper
{
    static LKDBHelper* db;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
#if TARGET_IPHONE_SIMULATOR//模拟器
        
    db = [[LKDBHelper alloc] initWithDBPath:DB_PATH];
        
#elif TARGET_OS_IPHONE//真机
        
    db = [[LKDBHelper alloc] initWithDBName:@"xx"];
        
#endif
        
    });
    return db;
}

//在类 初始化的时候
+(void)initialize
{
    //remove unwant property
    //比如 getTableMapping 返回nil 的时候   会取全部属性  这时候 就可以 用这个方法  移除掉 不要的属性
    [self removePropertyWithColumnName:@"error"];
}
```
比如要创建用户登录时保存用户登录返回字段的表，可以创建LoginModel继承baseModel，在登录表的.h文件中申明后台返回的字段，在.m文件中，实现方法如下：
```
-(id)initWithDict:(NSDictionary *)dict
{
    if (!dict) {
        return nil;
    }
    if(![[dict objectForKey:@"mobile"] isKindOfClass:[NSNull class]]){
        self.mobile = (NSString *)[dict objectForKey:@"mobile"];
    }
    if (![[dict objectForKey:@"_id"] isKindOfClass:[NSNull class]]) {
        self._id = (NSString *)[dict objectForKey:@"_id"];
    }
    if (![[dict objectForKey:@"user_name"] isKindOfClass:[NSNull class]]) {
        self.user_name = (NSString *)[dict objectForKey:@"user_name"];
    }
    return self;
}
//主键
+(NSString *)getPrimaryKey
{
    return @"_id";
}
```

##数据缓存
在请求后台接口成功后，初始化表结构，保存本地
```
    if(self.response.responseObject) {
        
        if (![[self.response.responseObject valueForKeyPath:@"data"] isKindOfClass:[NSNull class]]) {
            
            if ([self.response.responseObject valueForKeyPath:@"data.token"]) {
                NSDictionary *user_dict = [self.response.responseObject valueForKeyPath:@"data.user"];
                HZUser *user = [[HZUser alloc] initWithDict:user_dict];
                
                if ([user saveToDB]) {
                    NSLog(@"保存本地成功");
                }
        }
    }
```
如果更改了用户信息，需要请求其它接口,然后将返回的数据保存在user里表里，调用updateToDB方法更新表保存在本地。根据用户的id查找用户用以下的方法：
```
[HZUser searchSingleWithWhere:[NSString stringWithFormat:@"_id='%@'", GET_CURRENT_USER_ID_IN_NSUSER_DEFAULT] orderBy:@"_id" ];
```
如果要删除本地的表，需要调用deleteToDB的方法从本地删除。
##数据同步
例如项目中有一个保存商品属性的表，在新建商品的时候用上面的方法将商品表保存本地，在其他地方如果需要用到这个商品的属性去展示可以利用数据库语言通过商品的id查找到表然后展示。如果商品的信息在app交互上可以修改，就需要通过id查询到本地的表然后更新属性。一个商品对应一个库存，如果本地库存和后台的库存不一致时需要做数据同步，先取本地数据，然后从网上请求数据与本地比对，如果不一致就更新本地数据然后保存到本地。具体代码如下：
```
- (NSArray *)get_user_inventory {  

    if(self.response.responseObject) {       
        NSArray *arr = (NSArray *)[self.response.responseObject valueForKeyPath:@"data"] ;
        NSMutableArray *inventory_all_array = [NSMutableArray array];        
        //1.库存列表
        for (NSDictionary *dict in arr) {
            NSMutableDictionary *d = [NSMutableDictionary dictionaryWithDictionary:dict];            
            [d setObject:GET_CURRENT_USER_ID_IN_NSUSER_DEFAULT forKey:@"user_id"];            
            HZInventory *new_all = [[HZInventory alloc] initWithDict:d];            
            [d removeObjectForKey:@"inventory_array"]; 
           
            //获取已存在的数据模型
            HZInventory *oldAll = [HZInventory get_by_product_id:new_all.product_id]; 
           
            //从数据库删除已存在的模型
            if (oldAll) {
                [HZInventory deleteToDB:oldAll];
            }
                        
            if ([new_all saveToDB]) {
                [inventory_all_array addObject:new_all];
            };            
        return inventory_all_array;
    }
    return nil;
}
```
*以上就是项目中用到的数据缓存和同步，这里简单讲解一下使用方法，如果对你有用就mark一下吧！*
