---
layout: post
title: "NSPredicate"
date: 2014-05-17 20:42:11 +0800
comments: true
categories: 
- Objective-C
- 转载

---
本文转载自http://nshipster.cn/nspredicate/  

NSPredicate是一个Foundation类，它指定数据被获取或者过滤的方式。它的查询语言就像SQL的WHERE和正则表达式的交叉一样，提供了具有表现力的，自然语言界面来定义一个集合被搜寻的逻辑条件。  
<!--more-->
相比较抽象的谈论它，展示NSPredicate的使用方法更加容易，所以我们来重新审视[NSSortDescriptor](http://nshipster.com/nssortdescriptor/)中使用的示例数据集吧：  

索引 | 0 | 1 | 2 | 3 |
----|-------| ------|------- | ------------
名	| Alice	| Bob	| Charlie |	 Quentin
姓	| Smith	| Jones	| Smith	| Alberts
年龄	| 24	| 27	| 33	| 31


``` objc

@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@property NSNumber *age;
@end

@implementation Person

- (NSString *)description {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

@end

#pragma mark -

NSArray *firstNames = @[ @"Alice", @"Bob", @"Charlie", @"Quentin" ];
NSArray *lastNames = @[ @"Smith", @"Jones", @"Smith", @"Alberts" ];
NSArray *ages = @[ @24, @27, @33, @31 ];

NSMutableArray *people = [NSMutableArray array];
[firstNames enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    Person *person = [[Person alloc] init];
    person.firstName = firstNames[idx];
    person.lastName = lastNames[idx];
    person.age = ages[idx];
    [people addObject:person];
}];

NSPredicate *bobPredicate = [NSPredicate predicateWithFormat:@"firstName = 'Bob'"];
NSPredicate *smithPredicate = [NSPredicate predicateWithFormat:@"lastName = %@", @"Smith"];
NSPredicate *thirtiesPredicate = [NSPredicate predicateWithFormat:@"age >= 30"];

// ["Bob Jones"]
NSLog(@"Bobs: %@", [people filteredArrayUsingPredicate:bobPredicate]);

// ["Alice Smith", "Charlie Smith"]
NSLog(@"Smiths: %@", [people filteredArrayUsingPredicate:smithPredicate]);

// ["Charlie Smith", "Quentin Alberts"]
NSLog(@"30's: %@", [people filteredArrayUsingPredicate:thirtiesPredicate]);
``` 
##集合中使用NSPredicate

Foundation提供使用谓词（predicate）来过滤NSArray／NSMutableArray&NSSet／NSMutableSet的方法。  

不可变的集合，NSArray&NSSet，有可以通过评估接收到的predicate来返回一个不可变集合的方法`filteredArrayUsingPredicate:`和`filteredSetUsingPredicate:`。  

可变集合，NSMutableArray&NSMutableSet，可以使用方法`filterUsingPredicate:`，它可以通过运行接收到的谓词来移除评估结果为FALSE的对象。  

NSDictionary可以用谓词来过滤它的键和值（两者都为NSArray对象）。NSOrderedSet可以由过滤的NSArray或NSSet生成一个新的有序的集，或者NSMutableSet可以简单的`removeObjectsInArray:`，来传递通过_否定_predicate过滤的对象。  

##Core Data中使用NSPredicate

NSFetchRequest有一个predicate属性，它可以指定管理对象应该被获取的逻辑条件。谓词的使用规则在这里同样适用，唯一的区别在于，在管理对象环境中，谓词由持久化存储助理（persistent store coordinator）评估，而不像集合那样在内存中被过滤。  

##谓词语法

###替换

- %@是对值为字符串，数字或者日期的对象的替换值。  
- %K是key path的替换值。  

``` 
NSPredicate *ageIs33Predicate = [NSPredicate predicateWithFormat:@"%K = %@", @"age", @33];

// ["Charlie Smith"]
NSLog(@"Age 33: %@", [people filteredArrayUsingPredicate:ageIs33Predicate]);
``` 
- $VARIABLE_NAME是可以被`NSPredicate -predicateWithSubstitutionVariables:`替换的值。

``` 
NSPredicate *namesBeginningWithLetterPredicate = [NSPredicate predicateWithFormat:@"(firstName BEGINSWITH[cd] $letter) OR (lastName BEGINSWITH[cd] $letter)"];

// ["Alice Smith", "Quentin Alberts"]
NSLog(@"'A' Names: %@", [people filteredArrayUsingPredicate:[namesBeginningWithLetterPredicate predicateWithSubstitutionVariables:@{@"letter": @"A"}]]);
``` 
###基本比较

- =, ==：左边的表达式和右边的表达式相等。
- \>=, =>：左边的表达式大于或者等于右边的表达式。
- <=, =<：左边的表达式小于等于右边的表达式。
- \>：左边的表达式大于右边的表达式。
- <：左边的表达式小于右边的表达式。
- !=, <>：左边的表达式不等于右边的表达式。
- BETWEEN：左边的表达式等于右边的表达式的值或者介于它们之间。右边是一个有两个指定上限和下限的数值的数列（指定顺序的数列）。比如，1 BETWEEN { 0 , 33 }，或者$INPUT BETWEEN { $LOWER, $UPPER }。

###基本复合谓词

- AND, &&：逻辑与.
- OR, ||：逻辑或.
- NOT, !：逻辑非.

###字符串比较

*字符串比较在默认的情况下是区分大小写和音调的。你可以在方括号中用关键字符c和d来修改操作符以相应的指定不区分大小写和变音符号，比如firstname BEGINSWITH[cd] $FIRST_NAME。*

- BEGINSWITH：左边的表达式以右边的表达式作为开始。
- CONTAINS：左边的表达式包含右边的表达式。
- ENDSWITH：左边的表达式以右边的表达式作为结束。
- LIKE：左边的表达式等于右边的表达式：?和*可作为通配符，其中?匹配1个字符，*匹配0个或者多个字符。
- MATCHES：左边的表达式根据ICU v3（更多内容请查看[ICU User Guide for Regular Expressions](http://userguide.icu-project.org/strings/regexp)）的regex风格比较，等于右边的表达式。

###合计操作

####关系操作

- ANY，SOME：指定下列表达式中的任意元素。比如，ANY children.age < 18。
- ALL：指定下列表达式中的所有元素。比如，ALL children.age < 18。
- NONE：指定下列表达式中没有的元素。比如，NONE children.age < 18。它在逻辑上等于NOT (ANY ...)。
- IN：等于SQL的IN操作，左边的表达必须出现在右边指定的集合中。比如，name IN { 'Ben', 'Melissa', 'Nick' }。

####数组操作

- array[index]：指定数组中特定索引处的元素。
- array[FIRST]：指定数组中的第一个元素。
- array[LAST]：指定数组中的最后一个元素。
- array[SIZE]：指定数组的大小。

###布尔值谓词

- TRUEPREDICATE：结果始终为真的谓词。
- FALSEPREDICATE：结果始终为假的谓词。

##NSCompoundPredicate

我们见过与&或被用在谓词格式字符串中以创建复合谓词。然而，我们也可以用NSCompoundPredicate来完成同样的工作。  

例如，下列谓词是相等的：  

``` 
[NSCompoundPredicate andPredicateWithSubpredicates:@[[NSPredicate predicateWithFormat:@"age > 25"], [NSPredicate predicateWithFormat:@"firstName = %@", @"Quentin"]]];

[NSPredicate predicateWithFormat:@"(age > 25) AND (firstName = %@)", @"Quentin"];
``` 

虽然语法字符串文字更加容易输入，但是在有的时候，你需要结合现有的谓词。在那些情况下，你可以使用`NSCompoundPredicate -andPredicateWithSubpredicates:`&`-orPredicateWithSubpredicates:`。  

##NSComparisonPredicate

同样的，如果你在读过[上周的文章](http://nshipster.com/nsexpression/)之后发现你使用了太多的NSExpression的话，NSComparisonPredicate可以帮助你解决这个问题。  

就像NSCompoundPredicate一样，NSComparisonPredicate从子部件构建了一个NSPredicate－－在这种情况下，左侧和右侧都是NSExpression。 分析它的类的构造函数可以让我们一窥NSPredicate的格式字符串是如何解析的：  

``` 
+ (NSPredicate *)predicateWithLeftExpression:(NSExpression *)lhs
                             rightExpression:(NSExpression *)rhs
                                    modifier:(NSComparisonPredicateModifier)modifier
                                        type:(NSPredicateOperatorType)type
                                     options:(NSUInteger)options
``` 

####参数

- lhs：左边的表达式。
- rhs：右边的表达式。
- modifier：应用的修改符。（ANY或者ALL）
- type：谓词运算符类型。
- options：要应用的选项。没有选项的话则为0。                                                                          
                                 
###NSComparisonPredicate类型

``` 
enum {
   NSLessThanPredicateOperatorType = 0,
   NSLessThanOrEqualToPredicateOperatorType,
   NSGreaterThanPredicateOperatorType,
   NSGreaterThanOrEqualToPredicateOperatorType,
   NSEqualToPredicateOperatorType,
   NSNotEqualToPredicateOperatorType,
   NSMatchesPredicateOperatorType,
   NSLikePredicateOperatorType,
   NSBeginsWithPredicateOperatorType,
   NSEndsWithPredicateOperatorType,
   NSInPredicateOperatorType,
   NSCustomSelectorPredicateOperatorType,
   NSContainsPredicateOperatorType,
   NSBetweenPredicateOperatorType
};
typedef NSUInteger NSPredicateOperatorType;
``` 

###NSComparisonPredicate选项

- NSCaseInsensitivePredicateOption：不区分大小写的谓词。你通过在谓词格式字符串中加入后面带有[c]的字符串操作（比如，"NeXT" like[c] "next"）来表达这一选项。
- NSDiacriticInsensitivePredicateOption：忽视发音符号的谓词。你通过在谓词格式字符串中加入后面带有[d]的字符串操作（比如，"naïve" like[d] "naive"）来表达这一选项。
- NSNormalizedPredicateOption：表示待比较的字符串已经被预处理了。这一选项取代了NSCaseInsensitivePredicateOption和NSDiacriticInsensitivePredicateOption，旨在用作性能优化的选项。你可以通过在谓词格式字符串中加入后面带有[n]的字符串（比如，"WXYZlan" matches[n] ".lan"）来表达这一选项。
- NSLocaleSensitivePredicateOption：表明要使用<，<=，=，=>，> 作为比较的字符串应该使用区域识别的方式处理。你可以通过在<，<=，=，=>，>其中之一的操作符后加入[l]（比如，"straße" >[l] "strasse"）以便在谓词格式字符串表达这一选项。

##Block谓词

最后，如果你实在不愿意学习NSPredicate的格式语法，你也可以学学NSPredicate +predicateWithBlock:。  

``` 
NSPredicate *shortNamePredicate = [NSPredicate predicateWithBlock:^BOOL(id evaluatedObject, NSDictionary *bindings) {
            return [[evaluatedObject firstName] length] <= 5;
        }];

// ["Alice Smith", "Bob Jones"]
NSLog(@"Short Names: %@", [people filteredArrayUsingPredicate:shortNamePredicate]);
``` 
...好吧，虽然使用`predicateWithBlock:`是懒人的做法，但它也并不是一无是处。  

事实上，因为block可以封装任意的计算，所以有一个查询类是无法以NSPredicate格式字符串形式来表达的（比如对运行时被动态计算的值的评估）。而且当同一件事情可以用NSExpression结合自定义选择器来完成时，block为完成工作提供了一个方便的接口。  

重要提示：**由`predicateWithBlock:`生成的NSPredicate不能用于由SQLite存储库支持的Core Data数据的提取要求。**  

我知道我已经说过很多次了，可是NSPredicate真的是Cocoa的优势之一。其他语言的第三方库如果能有它一半的能力就已经很幸运了－－更别提标准库了。对于我们这些应用和框架开发者来说，有它作为标准组件使得我们在处理数据时有了很大的优势。  

和NSExpression一样，NSPredicate一直在提醒我们Foundation有多么好：它不仅仅十分有用，它精致的构架和设计也是我们写代码时灵感的来源。  