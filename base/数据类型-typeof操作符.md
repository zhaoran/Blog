本文介绍了typeof操作符的ES实现

typeof 操作符的实现比较接近系统底层，基本是按照下面的表格来实现的：

Type of val | Result
----|----
Undefined | "undefined"
Null | "object"
Boolean | "boolean"
Number | "number"
String | "string"
Symbol | "symbol"
Object（实现了内部方法[[call]]） | "function"
Object（普通对象并且没有实现内部方法[[call]]） | "object"
Object（标准外部对象并且没有实现内部方法[[call]]） | "object"
Object（非标准外部对象并且没有实现内部方法[[call]]） | 由具体实现而定，但不能是"undefined","boolean","number","string","symbol","function"

有几个需要注意的地方：
1. `typeof null === 'object'`，在Javascript的首版实现中，JavaScript的值是由类型标识和值组成的，对象的类型标识是0，而null做为空指针，在大部分平台的值也是0，
2. function 作为一种Obje

```c
JS_TypeOfValue(JSContext *cx, jsval v)
{
    JSType type = JSTYPE_VOID;
    JSObject *obj;
    JSObjectOps *ops;
    JSClass *clasp;

    CHECK_REQUEST(cx);
    if (JSVAL_IS_VOID(v)) {
	type = JSTYPE_VOID;
    } else if (JSVAL_IS_OBJECT(v)) {
	obj = JSVAL_TO_OBJECT(v);
	if (obj &&
	    (ops = obj->map->ops,
	     ops == &js_ObjectOps
	     ? (clasp = OBJ_GET_CLASS(cx, obj),
		clasp->call || clasp == &js_FunctionClass)
	     : ops->call != 0)) {
	    type = JSTYPE_FUNCTION;
	} else {
	    type = JSTYPE_OBJECT;
	}
    } else if (JSVAL_IS_NUMBER(v)) {
	type = JSTYPE_NUMBER;
    } else if (JSVAL_IS_STRING(v)) {
	type = JSTYPE_STRING;
    } else if (JSVAL_IS_BOOLEAN(v)) {
	type = JSTYPE_BOOLEAN;
    }
    return type;
}
```
```
JSType
js::TypeOfValue(const Value& v)
{
    if (v.isNumber())
        return JSTYPE_NUMBER;
    if (v.isString())
        return JSTYPE_STRING;
    if (v.isNull())
        return JSTYPE_OBJECT;
    if (v.isUndefined())
        return JSTYPE_VOID;
    if (v.isObject())
        return TypeOfObject(&v.toObject());
    if (v.isBoolean())
        return JSTYPE_BOOLEAN;
    MOZ_ASSERT(v.isSymbol());
    return JSTYPE_SYMBOL;
}

JSType
js::TypeOfObject(JSObject* obj)
{
    if (EmulatesUndefined(obj))
        return JSTYPE_VOID;
    if (obj->isCallable())
        return JSTYPE_FUNCTION;
    return JSTYPE_OBJECT;
}

JS_PUBLIC_API(JSType)
JS_TypeOfValue(JSContext* cx, HandleValue value)
{
    AssertHeapIsIdle(cx);
    CHECK_REQUEST(cx);
    assertSameCompartment(cx, value);
    return TypeOfValue(value);
}

#define TYPEOF(cx,v)    (v.isNull() ? JSTYPE_NULL : JS_TypeOfValue(cx,v))

JS_PUBLIC_API(bool)
JS_InstanceOf(JSContext* cx, HandleObject obj, const JSClass* clasp, CallArgs* args)
{
    AssertHeapIsIdle(cx);
    CHECK_REQUEST(cx);
#ifdef DEBUG
    if (args) {
        assertSameCompartment(cx, obj);
        assertSameCompartment(cx, args->thisv(), args->calleev());
    }
#endif
    if (!obj || obj->getJSClass() != clasp) {
        if (args)
            ReportIncompatibleMethod(cx, *args, Valueify(clasp));
        return false;
    }
    return true;
}
```
