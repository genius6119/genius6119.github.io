---
layout:     post
title:      SpringDataJPA自定义方法约定
subtitle:   -可用于mysql，es等
date:       2019-07-23
author:     Zwx
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Spring
---

- Spring Data 的另一个强大功能，是根据方法名称自动实现功能。

- 比如：你的方法名叫做：findByTitle，那么它就知道你是根据title查询，然后自动帮你完成，无需写实现类。

- 当然，方法名称要符合一定的约定：

    | Keyword        | Sample    |  
    | --------   | :-----  | 
    | And        | findByNameAndPrice      |  
    | Or        | findByNameOrPrice      |  
    | Is        | findByName      |  
    | Not        | findByNameNot      |  
    | Between        | findByPriceBetween      |  
    | LessThanEqual        | findByPriceLessThan      |  
    | GreaterThanEqual        | findByPriceGreaterThan
      |  
    | Before        | findByPriceBefore      |  
    | After        | findByPriceAfter      |  
    | Like        | findByNameLike      |  
    | StartingWith        | findByNameStartingWith      |  
    | EndingWith        | findByNameEndingWith      |  
    | Contains/Containing	        | findByNameContaining      |  
    | In        | findByNameIn(Collection<String>names)
      |  
    | NotIn        | findByNameNotIn(Collection<String>names)
      |  
    | Near        | findByStoreNear      |  
    | True        | findByAvailableTrue      |  
    | False        | findByAvailableFalse      |  
    | OrderBy        | findByAvailableTrueOrderByNameDesc      |  

    