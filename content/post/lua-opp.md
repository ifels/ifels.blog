+++
title = "Lua OOP 实现"
description = "Nothing special, but one post is boring."
date = "2014-09-02"
categories = [ "example"]
tags = [
    "lua"
]
+++

lua OOP 实现
-------------------------

```
function clone(object)
    local lookup_table = {}
    local function _copy(object)
        if type(object) ~= "table" then
            return object
        elseif lookup_table[object] then
            return lookup_table[object]
        end
        local new_table = {}
        lookup_table[object] = new_table
        for key, value in pairs(object) do
            new_table[_copy(key)] = _copy(value)
        end
        return setmetatable(new_table, getmetatable(object))
    end
    return _copy(object)
end

function class(classname, super)
    local superType = type(super)
    local cls

    if superType ~= "function" and superType ~= "table" then
        superType = nil
        super = nil
    end

    if superType == "function" or (super and super.__ctype == 1) then
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
        end

        cls.ctor    = function() end
        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end

    else
        -- inherited from Lua Object
        if super then
            cls = {}
            setmetatable(cls, {__index = super})
            cls.super = super
            if not cls.ctor then
                cls.ctor = function()
                    local parent = cls.super
                    while(parent) do
                        parent.ctor(self)
                        parent = parent.super
                    end
                end
            end
        else
            cls = {ctor = function() end}
        end

        cls.__cname = classname
        cls.__ctype = 2 -- lua
        cls.__index = cls

        function cls.new(...)
            local instance = setmetatable({}, cls)
            instance.class = cls
            instance:ctor(...)
            return instance
        end
    end

    return cls
end
```


使用示例
----------

```
Base = class("Base")

function Base:dump()
    print("Base.a = " .. tostring(self.a))
end

function Base:ctor()
    self.a = 1
    self.b = "b"
end

function Base:setA(a)
    self.a = a
end

Sub1 = class("Sub1", Base)

function Sub1:setA(a)
    self.a = a
end

function Sub1:dump()
    self.super.dump(self)
    print("Sub.a = " .. tostring(self.a))
end

--[[
function Sub1:ctor()
    self.super.ctor(self)
end
]]--

Sub2 = class("Sub2", Base)
function Sub2:setA(a)
    self.super.setA(self, a)
end

function Sub2:dump()
    self.super.dump(self)
    print("Sub2.a = " .. tostring(self.a))
end

function Sub2:ctor()
    self.super.ctor(self)
end

base1 = Base:new()
base2 = Base:new()
base1:setA(3)
base1:dump()
base2:dump()

print("----------")

sub1 = Sub1:new()
sub2 = Sub2:new()

sub1:dump()
sub2:dump()

print("----------")

sub1:setA(2)
sub2:setA(3)

sub1:dump()
sub2:dump()
```

注：调用父类方法时，使用self.spuer.method(self, ...), 不要使用self.spuer:method(...)。 这样可以保证父类和子类的self是同一个。  
  
参考自：[cocos2d-x](https://github.com/cocos2d/cocos2d-x/blob/v3/cocos/scripting/lua-bindings/script/extern.lua)

