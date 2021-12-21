## go版本 protobuf源码学习

### 先只看序列化相关
忽略所有oneof的处理 先看基础的

### protobuf版本
`github.com/golang/protbuf`

### marshal流程
1. 使用XXX_Size()和XXX_Marshal() 在生成的pb文件中生成好的
```go
func Marshal(pb Message) ([]byte, error) {
	if m, ok := pb.(newMarshaler); ok {
		siz := m.XXX_Size()
		b := make([]byte, 0, siz)
		return m.XXX_Marshal(b, false)
	}
	if m, ok := pb.(Marshaler); ok {
		// If the message can marshal itself, let it do it, for compatibility.
		// NOTE: This is not efficient.
		return m.Marshal()
	}
	// in case somehow we didn't generate the wrapper
	if pb == nil {
		return nil, ErrNil
	}
	var info InternalMessageInfo
	siz := info.Size(pb)
	b := make([]byte, 0, siz)
	return info.Marshal(b, pb, false)
}
```
2. pb文件中的方法
```go
func (m *RoleData) XXX_Unmarshal(b []byte) error {
	return xxx_messageInfo_RoleData.Unmarshal(m, b)
}
func (m *RoleData) XXX_Marshal(b []byte, deterministic bool) ([]byte, error) {
	return xxx_messageInfo_RoleData.Marshal(b, m, deterministic)
}
func (m *RoleData) XXX_Merge(src proto.Message) {
	xxx_messageInfo_RoleData.Merge(m, src)
}
func (m *RoleData) XXX_Size() int {
	return xxx_messageInfo_RoleData.Size(m)
}
func (m *RoleData) XXX_DiscardUnknown() {
	xxx_messageInfo_RoleData.DiscardUnknown(m)
}
var xxx_messageInfo_RoleData proto.InternalMessageInfo
```
3. InternalMessageInfo中的marshal
```go
func (a *InternalMessageInfo) Marshal(b []byte, msg Message, deterministic bool) ([]byte, error) {
	u := getMessageMarshalInfo(msg, a)
	ptr := toPointer(&msg)
	if ptr.isNil() {
		// We get here if msg is a typed nil ((*SomeMessage)(nil)),
		// so it satisfies the interface, and msg == nil wouldn't
		// catch it. We don't want crash in this case.
		return b, ErrNil
	}
	return u.marshal(b, ptr, deterministic)
}
```
4. 获得marshalinfo

marshalinfo的结构
```go
type marshalInfo struct {
	typ          reflect.Type
	fields       []*marshalFieldInfo
	unrecognized field                      // offset of XXX_unrecognized
	extensions   field                      // offset of XXX_InternalExtensions
	v1extensions field                      // offset of XXX_extensions
	sizecache    field                      // offset of XXX_sizecache
	initialized  int32                      // 0 -- only typ is set, 1 -- fully initialized
	messageset   bool                       // uses message set wire format
	hasmarshaler bool                       // has custom marshaler
	sync.RWMutex                            // protect extElems map, also for initialization
	extElems     map[int32]*marshalElemInfo // info of extension elements
}

```
```go
func getMessageMarshalInfo(msg interface{}, a *InternalMessageInfo) *marshalInfo {
	// u := a.marshal, but atomically.
	// We use an atomic here to ensure memory consistency.
	u := atomicLoadMarshalInfo(&a.marshal)
	if u == nil {
		// Get marshal information from type of message.
		t := reflect.ValueOf(msg).Type()
		if t.Kind() != reflect.Ptr {
			panic(fmt.Sprintf("cannot handle non-pointer message type %v", t))
		}
		u = getMarshalInfo(t.Elem())
		// Store it in the cache for later users.
		// a.marshal = u, but atomically.
		atomicStoreMarshalInfo(&a.marshal, u)
	}
	return u
}
可是发现这里如果没有只是 把marshalinfo放到了一个全局的map中缓存 实际上marshalInfo中的字段都是哪初始化的呢
```
5. size()函数
```go
func (u *marshalInfo) size(ptr pointer) int {
	if atomic.LoadInt32(&u.initialized) == 0 {
		u.computeMarshalInfo()
	}
```
在size中如果info没有被初始化 那么调用`computeMarshalInfo`初始化 而且由于是放在`全局的map`中的 只是第一次调用时会`初始化一次`
```go
func (u *marshalInfo) computeMarshalInfo() {
	u.Lock()
	defer u.Unlock()
	if u.initialized != 0 { // non-atomic read is ok as it is protected by the lock
		return
	}

	t := u.typ
	u.unrecognized = invalidField
	u.extensions = invalidField
	u.v1extensions = invalidField
	u.sizecache = invalidField

	n := t.NumField()

	// deal with XXX fields first
  // filter XXX字段 code ignore

	// normal fields
	fields := make([]marshalFieldInfo, n) // batch allocation
	u.fields = make([]*marshalFieldInfo, 0, n)
	for i, j := 0, 0; i < t.NumField(); i++ {
		f := t.Field(i)

		if strings.HasPrefix(f.Name, "XXX_") {
			continue
		}
		field := &fields[j]
		j++
		field.name = f.Name
		u.fields = append(u.fields, field)
		if f.Tag.Get("protobuf_oneof") != "" {
			field.computeOneofFieldInfo(&f, oneofImplementers)
			continue
		}
		if f.Tag.Get("protobuf") == "" {
			// field has no tag (not in generated message), ignore it
			u.fields = u.fields[:len(u.fields)-1]
			j--
			continue
		}
		field.computeMarshalFieldInfo(&f)
	}

	// fields are marshaled in tag order on the wire.
	sort.Sort(byTag(u.fields))
	atomic.StoreInt32(&u.initialized, 1)
}
```

computeMarshalFieldInfo处理每个字段
```go
func (fi *marshalFieldInfo) computeMarshalFieldInfo(f *reflect.StructField) {
	// parse protobuf tag of the field.
	// tag has format of "bytes,49,opt,name=foo,def=hello!"
	tags := strings.Split(f.Tag.Get("protobuf"), ",")
	if tags[0] == "" {
		return
	}
	tag, err := strconv.Atoi(tags[1])
	if err != nil {
		panic("tag is not an integer")
	}
	wt := wiretype(tags[0])
	if tags[2] == "req" {
		fi.required = true
	}
	fi.setTag(f, tag, wt)
	fi.setMarshaler(f, tags)
}
```

```go
func (fi *marshalFieldInfo) setTag(f *reflect.StructField, tag int, wt uint64) {
	fi.field = toField(f)
	fi.wiretag = uint64(tag)<<3 | wt
	fi.tagsize = SizeVarint(uint64(tag) << 3)
}
```
可以说明wiretype 占低3位 其他为tag
tagsize则是一个varint SizeVarint会根据值的大小来返回占用字节数

```go
func (fi *marshalFieldInfo) setMarshaler(f *reflect.StructField, tags []string) {
	switch f.Type.Kind() {
	case reflect.Map:
		// map field
		fi.isPointer = true
		fi.sizer, fi.marshaler = makeMapMarshaler(f)
		return
	case reflect.Ptr, reflect.Slice:
		fi.isPointer = true
	}
	fi.sizer, fi.marshaler = typeMarshaler(f.Type, tags, true, false)
}
```
可以看出map有单独的marshaler和sizer
map存储时还是一个list repeated,它还需要处理map的key和value

```go
// typeMarshaler returns the sizer and marshaler of a given field.
// t is the type of the field.
// tags is the generated "protobuf" tag of the field.
// If nozero is true, zero value is not marshaled to the wire.
// If oneof is true, it is a oneof field.
func typeMarshaler(t reflect.Type, tags []string, nozero, oneof bool) (sizer, marshaler) {
}
```
- nozero参数 可以让mashal时zerovalue忽略掉
- 此函数会根据各种参数 来觉得 到底使用哪一种marshler

至此所有marshaler的func都确定了 
在后续真正marshal时会调用调用每个字段的marshaler

6. pb文件中的tag含义 在代码中是有用到的 如果没有tag在marshal时会被忽略的
```go
	Id                   uint32                `protobuf:"varint,1,opt,name=Id,proto3" json:"Id,omitempty"`
	AccountId            string                `protobuf:"bytes,2,opt,name=AccountId,proto3" json:"AccountId,omitempty"`
	Currency             map[int32]int64       `protobuf:"bytes,23,rep,name=Currency,proto3" json:"Currency,omitempty" protobuf_key:"varint,1,opt,name=key,proto3" protobuf_val:"varint,2,opt,name=value,proto3"`
```
- varint bytes这些是标明 wiretype 即编码后的类型
- 1,2,23这些是tag 即proto中定义的字段的索引
- protobuf_key proto_buf_value是 map的key和value的 tag定义

### 编码后的格式
Protobuf消息序列化之后，会产生二进制数据。

这些数据（精确到bit）按照含义不同，可以划分为6个部分：`MSB` flag、tag、编码后数据类型（`wire type`）、长度（`length`）、字段值（`value`）、以及填充（`padding`）
- msb占1位
- wire type 3位


### unmarshal流程
```
func Unmarshal(buf []byte, pb Message) error {
	pb.Reset()
	if u, ok := pb.(newUnmarshaler); ok {
		return u.XXX_Unmarshal(buf)
	}
	if u, ok := pb.(Unmarshaler); ok {
		return u.Unmarshal(buf)
	}
	return NewBuffer(buf).Unmarshal(pb)
}
```

反序列化的流程基本上一致 有一些其他的细节优化
```go
type unmarshalInfo struct {
	typ reflect.Type // type of the protobuf struct

	// 0 = only typ field is initialized
	// 1 = completely initialized
	initialized     int32
	lock            sync.Mutex                    // prevents double initialization
	dense           []unmarshalFieldInfo          // fields indexed by tag #
	sparse          map[uint64]unmarshalFieldInfo // fields indexed by tag #
	reqFields       []string                      // names of required fields
	reqMask         uint64                        // 1<<len(reqFields)-1
	unrecognized    field                         // offset of []byte to put unrecognized data (or invalidField if we should throw it away)
	extensions      field                         // offset of extensions field (of type proto.XXX_InternalExtensions), or invalidField if it does not exist
	oldExtensions   field                         // offset of old-form extensions field (of type map[int]Extension)
	extensionRanges []ExtensionRange              // if non-nil, implies extensions field is valid
	isMessageSet    bool                          // if true, implies extensions field is valid
}
```

dense是一个slice 如果字段个数比较少 或者 字段索引值比较小 会直接保存到slice中 unmarshal的时候找unmarshaler更快
否则使用sparse的map
```go
func (u *unmarshalInfo) setTag(tag int, field field, unmarshal unmarshaler, reqMask uint64, name string) {
	i := unmarshalFieldInfo{field: field, unmarshal: unmarshal, reqMask: reqMask, name: name}
	n := u.typ.NumField()
	if tag >= 0 && (tag < 16 || tag < 2*n) { // TODO: what are the right numbers here?
		// 这里是填充padding
		for len(u.dense) <= tag {
			u.dense = append(u.dense, unmarshalFieldInfo{})
		}
		u.dense[tag] = i
		return
	}
	if u.sparse == nil {
		u.sparse = map[uint64]unmarshalFieldInfo{}
	}
	u.sparse[uint64(tag)] = i
}
```

### String()返回

### 个人感受
1. 序列化看起来又用reflect又各种转来转去 看似耗时 但实际上 序列化不就是在用cpu换内存嘛
2. 所以为什么不能开index值 因为marshal的时候会把索引值当成tag序列化进去 
   - 这样添加新的索引值并不会有问题 因为新的索引值 没有marshal的数据 
   - 删除了索引值 也会在unmarshal的时候因为没有找到对应字段的unmarshaler而skip

### 参考
[csdn](https://blog.csdn.net/zxhoo/article/details/53228303)
