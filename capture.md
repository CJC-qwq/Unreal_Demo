# 照相与物体投影系统详细技术解析

## 一、系统概述

该系统实现了一个创新性的游戏机制，允许玩家通过拍照捕获游戏世界中的特定物体，然后将这些物体通过"投影"的方式重建为实体。系统的核心是基于UE4的程序化网格技术（Procedural Mesh），通过对物体进行采样、切片和重建来实现物体的捕获和复制。

## 二、核心组件结构

### 1. 相机系统 (A_Camera)

`A_Camera`是系统的核心，负责捕获和投影物体：

- **范围检测**: 通过`Is_Inside()`函数判断物体是否在相机视野范围内
- **物体捕获**: `Capture_Photo()`函数获取视野内物体信息
- **视野扫描**: `Get_Items_In_Frame()`使用射线检测识别视野内物体
- **物体切割**: `Cut_Items_In_Frame()`在捕获时对物体进行切割
- **物体销毁**: `Destroy_Actors_In_Inside()`清理不需要的网格部分
- **物体投影**: `Place_New_Items()`根据照片信息重建物体

### 2. 视图查找器组件 (A_ViewFinder_Component)

这是附加在玩家角色上的组件，处理玩家输入和相机交互：

- **输入处理**: 绑定玩家输入动作到相应函数
- **相机控制**: 提供`Focus_on_Photo()`和`Lose_Focus_On_Photo()`功能
- **照片捕获**: 触发`Capture_Photo()`开始拍照流程
- **照片放置**: `Place_Photo()`将拍摄的照片放置在游戏世界中

### 3. 照片类 (A_Photo)

存储并展示已捕获照片的类：

- **静态网格**: 使用`UStaticMeshComponent`展示照片
- **照片数据**: 存储`PhotoInformation`数组，包含所有捕获的物体信息

### 4. 可处理基类 (A_Procedural_Base)

所有可被拍摄和投影物体的基类：

- **网格设置**: `SetUp_Mesh()`初始化程序化网格
- **切片处理**: `Slice_Mesh()`实现网格沿平面切割
- **销毁处理**: `Destory_Mesh()`销毁不需要的网格部分
- **有效点判定**: `Determine_Valid_Points()`确定有效的物体点
- **照片切片重建**: `Slice_Based_on_Photo()`根据照片信息重建物体
- **相对位置计算**: `Get_Relative_Bounds_Origin()`计算组件相对边界位置

### 5. 物理可处理类 (A_Procedural_Physics)

具有物理属性的可处理物体类，继承自`A_Procedural_Base`，添加了物理模拟能力。

### 6. 数据结构 (A_Struct)

定义了系统使用的核心数据结构：

- **照片信息结构体** (FST_Photoinfo): 存储捕获物体的完整信息，包括：
  - 静态网格引用
  - 材质列表
  - 是否启用物理
  - 变换信息（位置、旋转、缩放）
  - 切片信息（法线和位置映射）
  - 有效点列表
  - 旧切割数量记录

## 三、功能完整工作流程

### 1. 拍照流程详解

#### 初始化阶段：
1. 玩家通过输入触发`A_ViewFinder_Component`的`Capture_Photo()`函数
2. 该函数调用相机的`Capture_Photo()`方法开始捕获过程

#### 物体检测阶段：
3. 相机调用`Get_Items_In_Frame()`函数扫描视野内物体：
   - 通过`GetActorTransform()`获取相机的变换矩阵
   - 计算切割向量`L_Cut_Vector`，初始为(-0.203, -0.979, 0.0)
   - 在一个循环中，从不同角度发射射线
   - 使用`LineTraceMultiByChannel()`进行射线检测
   - 记录命中的`A_Procedural_Base`类物体
   - 循环中通过改变`L_Right_Alpha`和`L_Up_Alpha`值覆盖整个视野
   - 在四个不同的阶段（由`L_Switch`控制）改变切割向量方向

#### 物体信息提取阶段：
4. 对于每个检测到的物体：
   - 获取物体变换（位置、旋转、缩放）
   - 将世界空间坐标转换为相机本地空间坐标
   - 提取物体的`PhotoInformation`结构
   - 更新结构中的变换信息
   - 将更新后的信息添加到照片信息数组中
   - 调用物体的`Clear_Extra_Slice()`清理多余切片信息

#### 照片生成阶段：
5. 完成信息收集后，生成包含所有捕获物体信息的照片数据

### 2. 物体切割流程详解

1. 在拍照过程中，`Cut_Items_In_Frame()`函数执行物体切割：
   - 设置初始切割向量(-0.203, -0.979, 0.0)
   - 循环中发射射线检测物体
   - 对每个命中点，获取物体实例并记录到`L_AlreadyCutActors`
   - 检查物体是否已处理，避免重复切割
   - 对未处理物体调用`Slice_Mesh()`进行切割
   - 切割后保存新生成的网格部分
   - 同拍照过程一样，通过四阶段扫描覆盖整个视野

2. `Slice_Mesh()`函数实现具体切割操作：
   - 如需记录切割信息，将平面法线和位置添加到`PhotoInformation.Slices`
   - 递增`Old_Cuts`计数器
   - 对物体的每个程序化网格组件执行切割
   - 使用`UKismetProceduralMeshLibrary::SliceProceduralMesh()`进行实际切割
   - 保存切割产生的新网格部分
   - 将新网格添加到物体的网格组件列表

3. 完成切割后，调用`Destroy_Actors_In_Inside()`销毁不需要的部分：
   - 获取所有`A_Procedural_Base`类实例
   - 检查每个实例是否在相机内部
   - 对在内部的实例，创建新的程序化网格组件
   - 调用物体的`Destory_Mesh()`方法销毁这些组件
   - 处理切割产生的额外网格部分
   - 对保留的物体部分调用`Determine_Valid_Points()`确定有效点

### 3. 投影重建流程详解

1. 当玩家触发投影时，调用`Place_New_Items()`函数：
   - 遍历照片信息数组中的每个物体信息
   - 提取物体的位置、旋转和比例信息
   - 将相机本地坐标转换回世界坐标

2. 根据物体类型生成相应实例：
   - 如果`Physics`为true，生成`AA_Procedural_Physics`实例
   - 否则生成`AA_Procedural_Base`实例
   - 设置生成物体的所有者为当前相机
   - 复制照片信息到新物体
   - 设置新物体的变换矩阵

3. 调用`Slice_Based_on_Photo()`根据照片信息重建物体形状：
   - 首先检查`Old_Cuts`是否大于0，确定是否有切片信息
   - 提取切片信息中的平面法线和位置
   - 对每个切片平面，调用`Slice_Mesh()`进行切割
   - 删除不符合原始形状的部分
   - 应用所有记录的切片操作
   - 对物理物体启用物理模拟

4. 完成重建后，对物体进行清理和优化：
   - 调用`Determine_Valid_Points()`确定有效点
   - 设置网格组件移动性为`Movable`
   - 如需要，启用物理模拟

## 四、关键算法详解

### 1. 视野内物体检测算法

`Is_Inside()`函数判断物体是否在相机视野内：

```cpp
void AA_Camera::Is_Inside(FVector Location, bool& Return_Value, double L_Alpha_Distance, double L_Relative_Y, double L_Relative_Z)
{
    // 将世界坐标转换为相机本地坐标
    FTransform ActorTransform = GetActorTransform();
    FVector Return_Location = ActorTransform.InverseTransformPosition(Location);
    double Out_X = Return_Location.X;
    double Out_Y = Return_Location.Y;
    double Out_Z = Return_Location.Z;

    // 只检查前方物体(X>0)
    if (Out_X >= 0)
    {
        // 计算深度比例和相对位置
        L_Alpha_Distance = Out_X / Range;
        L_Relative_Y = Out_Y;
        L_Relative_Z = Out_Z;
        
        // 根据深度比例计算边界
        double Lerp_Return = FMath::Lerp(10.5, 1050, L_Alpha_Distance);
        double Position = Lerp_Return;
        double Negative = Lerp_Return * -1;
        
        // 判断是否在边界内
        Return_Value = (L_Relative_Y < Position) && (L_Relative_Z < Position) && 
                       (L_Relative_Y > Negative) && (L_Relative_Z < Negative);
    }
    else
    {
        Return_Value = false;
    }
}
```

该算法创建了一个根据深度动态扩大的截锥体，远处的物体需要在更大的Y/Z范围内才被认为在视野内。

### 2. 网格切割算法

`Slice_Mesh()`函数实现了网格沿平面切割：

```cpp
void AA_Procedural_Base::Slice_Mesh(bool Add_Cuts, FVector Plane_Position, FVector Plane_Normal, 
                                    TArray<UProceduralMeshComponent*>& NewParam, 
                                    TArray<UProceduralMeshComponent*> L_Returan_Meshes, 
                                    TArray<UProceduralMeshComponent*> L_Current_Meshes, 
                                    FTransform L_TransForm)
{
    // 获取物体当前变换
    L_TransForm = GetActorTransform();

    // 记录切割平面信息
    if (Add_Cuts)
    {
        PhotoInformation.Slices.Add(
            GetActorTransform().InverseTransformVectorNoScale(Plane_Normal), 
            GetActorTransform().TransformPosition(Plane_Position)
        );
        PhotoInformation.Old_Cuts++;
    }

    // 对所有网格组件执行切割
    L_Current_Meshes = Procedural_mesh_comp;
    for (UProceduralMeshComponent* Array_Element : L_Current_Meshes)
    {
        if (Array_Element->GetOwner() == this)
        {
            // 创建新网格存储切割结果
            UProceduralMeshComponent* Out_Other_Half_Proc_Mesh;
            UMaterialInterface* Material = NewObject<UMaterialInterface>();
            
            // 执行实际切割
            UKismetProceduralMeshLibrary::SliceProceduralMesh(
                Array_Element, Plane_Position, Plane_Normal, true, 
                Out_Other_Half_Proc_Mesh, EProcMeshSliceCapOption::UseLastSectionForCap, Material
            );
            
            // 存储切割结果
            L_Returan_Meshes.Add(Out_Other_Half_Proc_Mesh);
            Procedural_mesh_comp.Add(Out_Other_Half_Proc_Mesh);
        }
    }
    
    // 返回新创建的网格
    NewParam = L_Returan_Meshes;
}
```

该算法使用UE4的程序化网格库进行切割，并存储切割平面信息以便后续重建。

### 3. 物体重建算法

`Slice_Based_on_Photo()`函数根据存储的切片信息重建物体：

```cpp
void AA_Procedural_Base::Slice_Based_on_Photo(FVector L_Normal_Relative, FVector L_Position_Relative, 
                                             TArray<UProceduralMeshComponent*> L_CutsToDelete, 
                                             FTransform L_Transform)
{
    // 获取当前变换
    L_Transform = GetActorTransform();
    
    // 应用之前记录的切片
    if (PhotoInformation.Old_Cuts != 0)
    {
        TArray<FVector> keys;
        PhotoInformation.Slices.GetKeys(keys);
        
        // 对每个记录的切片进行处理
        for (int i = 0; i < keys.Num(); i++)
        {
            L_Normal_Relative = keys[i];
            L_Position_Relative = *PhotoInformation.Slices.Find(L_Normal_Relative);
            
            // 只处理到旧切割数量
            if (i == PhotoInformation.Old_Cuts)
                break;
                
            // 执行切割
            TArray<UProceduralMeshComponent*> NewParam;
            TArray<UProceduralMeshComponent*> L_Returan_Meshes;
            TArray<UProceduralMeshComponent*> L_Current_Meshes;
            FTransform L_TransForm;
            
            Slice_Mesh(false, 
                      L_Transform.TransformPosition(L_Position_Relative), 
                      L_Transform.TransformVectorNoScale(L_Normal_Relative), 
                      NewParam, L_Returan_Meshes, L_Current_Meshes, L_TransForm);
        }
        
        // 删除不符合原始形状的部分
        for (UProceduralMeshComponent* Array_Element : Procedural_mesh_comp)
        {
            if (Array_Element->GetOwner() == this)
            {
                FVector value_in = Get_Relative_Bounds_Origin(Array_Element);
                bool return_value;
                Almost_Contains(value_in, 0.5, return_value);
                
                // 如果不在有效点附近，销毁该部分
                if (!return_value)
                {
                    UProceduralMeshComponent* L_ProceduralMesh;
                    Destory_Mesh(Array_Element, L_ProceduralMesh);
                }
            }
        }
    }
    
    // 应用新切片(在拍照后添加的切片)
    TArray<FVector> keys;
    PhotoInformation.Slices.GetKeys(keys);
    
    for (int i = 0; i < keys.Num(); i++)
    {
        L_Normal_Relative = keys[i];
        L_Position_Relative = *PhotoInformation.Slices.Find(L_Normal_Relative);
        
        // 处理新切割
        if (i >= PhotoInformation.Old_Cuts)
        {
            TArray<UProceduralMeshComponent*> NewParam;
            TArray<UProceduralMeshComponent*> L_Returan_Meshes;
            TArray<UProceduralMeshComponent*> L_Current_Meshes;
            FTransform L_TransForm;
            
            Slice_Mesh(false,
                      GetActorTransform().TransformPosition(L_Position_Relative), 
                      GetActorTransform().TransformVectorNoScale(L_Normal_Relative), 
                      NewParam, L_Returan_Meshes, L_Current_Meshes, L_TransForm);
                      
            PhotoInformation.Old_Cuts++;
        }
    }
    
    // 清理相机内部的网格
    for (UProceduralMeshComponent* Array_Element : Procedural_mesh_comp)
    {
        if (Array_Element->GetOwner() == this && IsValid(A_Camera))
        {
            FVector Origin;
            FVector Box_Extent;
            float Sphere_Radius = 0;
            bool Return_Value = false;
            
            UKismetSystemLibrary::GetComponentBounds(Array_Element, Origin, Box_Extent, Sphere_Radius);
            A_Camera->Is_Inside(Origin, Return_Value, 0, 0, 0);
            
            // 如果不在相机内部，标记为删除
            if (!Return_Value)
            {
                L_CutsToDelete.Add(Array_Element);
            }
        }
    }
    
    // 删除标记的网格
    for (UProceduralMeshComponent* Array_Element : L_CutsToDelete)
    {
        UProceduralMeshComponent* L_ProceduralMesh;
        Destory_Mesh(Array_Element, L_ProceduralMesh);
    }
    
    // 确定有效点并启用物理
    Determine_Valid_Points(false);
    
    for (UProceduralMeshComponent* Array_Element : Procedural_mesh_comp)
    {
        Array_Element->SetMobility(EComponentMobility::Movable);
        Array_Element->SetSimulatePhysics(true);
    }
}
```

该算法通过应用存储的切片信息，先重建物体原始形状，然后应用新切片，最后删除相机内部的网格以创建完整的物体。

### 4. 扫描线算法

`Get_Items_In_Frame()`和`Cut_Items_In_Frame()`使用类似的扫描线算法覆盖相机视野：

```cpp
while (L_Count < (FMath::RoundToInt(2 / L_Size_Interval) * 4))
{
    // 计算射线起点和终点
    FVector Start = GetActorLocation() + 
                   (GetActorRightVector() * (L_Right_Alpha * 10.45)) + 
                   (GetActorUpVector() * (L_Up_Alpha * 10.45));
                   
    FVector End = (GetActorLocation() + (GetActorForwardVector() * Range)) + 
                 (GetActorRightVector() * (L_Right_Alpha * 1045)) + 
                 (GetActorUpVector() * (L_Up_Alpha * 1045));
    
    // 发射射线
    TArray<FHitResult> Out_Hits;
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);
    GetWorld()->LineTraceMultiByChannel(Out_Hits, Start, End, ECC_GameTraceChannel1, QueryParams);
    
    // 处理命中结果...
    
    // 根据当前阶段更新扫描参数
    L_Count++;
    switch (L_Switch)
    {
    case 0: // 从上到下扫描
        L_Up_Alpha = L_Up_Alpha - L_Size_Interval;
        if (abs((float)(L_Up_Alpha - (-1.0))) < 0.001)
        {
            L_Switch++;
            L_Cut_Vector = GetActorTransform().TransformVectorNoScale(FVector(-0.203, 0.0, -0.979));
            L_AlreadyCutActors_Modified.Empty();
        }
        break;
    case 1: // 从左到右扫描
        L_Right_Alpha = L_Right_Alpha + L_Size_Interval;
        if (abs((float)(L_Right_Alpha - 1.0)) < 0.001)
        {
            L_Switch++;
            L_Cut_Vector = GetActorTransform().TransformVectorNoScale(FVector(-0.203, 0.979, 0.0));
            L_AlreadyCutActors_Modified.Empty();
        }
        break;
    case 2: // 从下到上扫描
        L_Up_Alpha = L_Up_Alpha + L_Size_Interval;
        if (abs((float)(L_Up_Alpha - 1.0)) < 0.001)
        {
            L_Switch++;
            L_Cut_Vector = GetActorTransform().TransformVectorNoScale(FVector(-0.203, 0.0, 0.979));
            L_AlreadyCutActors_Modified.Empty();
        }
        break;
    case 3: // 从右到左扫描
        L_Right_Alpha = L_Right_Alpha - L_Size_Interval;
        break;
    }
}
```

该算法通过在四个方向上系统地发射射线，创建一个完整的扫描网格，确保视野范围内的所有物体都能被检测到。

## 五、数据流转详解

### 1. 照片信息结构

`FST_Photoinfo`是整个系统的核心数据结构：

```cpp
struct FST_Photoinfo
{
    // 物体的静态网格
    TObjectPtr<UStaticMesh> StaticMesh;
    
    // 物体的材质列表
    TArray<UMaterialInterface*> Materials;
    
    // 是否启用物理
    bool Physics;
    
    // 物体的变换信息(位置、旋转、缩放)
    FTransform Transform;
    
    // 切片平面信息：<法线, 位置>映射
    TMap<FVector, FVector> Slices;
    
    // 有效点列表
    TArray<FVector> Valid_Points;
    
    // 旧切割数量
    int32 Old_Cuts;
};
```

这个结构体存储了重建物体所需的所有信息：

- 静态网格和材质定义了物体的外观
- 变换信息定义了物体在相机空间中的位置
- 切片信息定义了物体被切割的平面
- 有效点用于验证重建物体的准确性
- 旧切割数量用于区分原始切片和新添加的切片

### 2. 数据转换流程

1. **捕获阶段**:
   - 从世界空间获取物体信息
   - 转换为相机本地空间坐标
   - 存储到`FST_Photoinfo`结构
   - 记录在`AA_Photo`实例中

2. **投影阶段**:
   - 从`FST_Photoinfo`读取物体信息
   - 转换回世界空间坐标
   - 创建新的物体实例
   - 应用存储的切片信息重建形状
   - 设置物理属性

### 3. 坐标系转换

系统中的坐标转换是关键要素，主要包括：

- **世界空间→相机空间**：使用`ActorTransform.InverseTransformPosition()`
- **相机空间→世界空间**：使用`ActorTransform.TransformPosition()`
- **向量方向转换**：使用`TransformVectorNoScale()`/`InverseTransformVectorNoScale()`

这些转换确保物体在捕获和重建过程中保持正确的位置和方向。

## 六、组件状态与生命周期

### 1. 程序化网格组件管理

物体的程序化网格组件通过以下方式管理：

- `SetUp_Mesh()`：初始化时从静态网格创建程序化网格
- `Procedural_mesh_comp`数组：跟踪所有网格部分
- `Slice_Mesh()`：切割网格并添加新部分
- `Destory_Mesh()`：删除不需要的网格部分
- `Clear_Small_Cuts()`：清理过小的网格片段

### 2. 物体生命周期

1. **创建阶段**：
   - 在`AA_Procedural_Base`构造函数中创建基本组件
   - 在`BeginPlay()`中初始化网格组件数组
   - 在`OnConstruction()`中调用`SetUp_Mesh()`设置网格

2. **捕获阶段**：
   - 物体被检测到在相机视野内
   - 物体信息被提取到`PhotoInformation`结构
   - 物体可能被切割以确定准确形状

3. **投影阶段**：
   - 创建新物体实例
   - 应用所有存储的切片信息
   - 删除不符合原始形状的部分
   - 启用物理模拟（如需要）

4. **销毁阶段**：
   - 当所有网格组件被移除后，调用`Destroy()`销毁物体
   - 通过`Destory_Mesh()`清理单个网格组件

## 七、技术实现细节

### 1. 射线检测系统

系统使用UE4的射线检测系统检测视野内物体：

```cpp
GetWorld()->LineTraceMultiByChannel(Out_Hits, Start, End, ECC_GameTraceChannel1, QueryParams);
```

这种方法允许检测所有位于射线路径上的物体，而不仅仅是第一个命中的物体。

### 2. 程序化网格切割

系统使用UE4的程序化网格库进行网格切割：

```cpp
UKismetProceduralMeshLibrary::SliceProceduralMesh(
    Array_Element, Plane_Position, Plane_Normal, true, 
    Out_Other_Half_Proc_Mesh, EProcMeshSliceCapOption::UseLastSectionForCap, Material
);
```

此函数沿指定平面切割网格，并创建切割面的封口，确保网格保持封闭。

### 3. 物理模拟

对于启用物理的物体，系统在重建后启用物理模拟：

```cpp
Array_Element->SetMobility(EComponentMobility::Movable);
Array_Element->SetSimulatePhysics(true);
```

这允许重建的物体与游戏世界中的其他物体交互。

### 4. 动态边界计算

系统使用`UKismetSystemLibrary::GetComponentBounds()`获取物体组件的边界信息：

```cpp
UKismetSystemLibrary::GetComponentBounds(Component, Origin, Box_Extent, Sphere_Radius);
```

这些边界信息用于判断物体是否在相机内部，以及删除过小的网格片段。

## 八、输入处理与用户交互

视图查找器组件通过Enhanced Input系统处理玩家输入：

```cpp
void UA_ViewFinder_Component::BeginPlay()
{
    // 获取玩家控制器
    APlayerController* PlayerController = GetWorld()->GetFirstPlayerController();
    if (PlayerController)
    {
        // 添加输入映射上下文
        UEnhancedInputLocalPlayerSubsystem* Subsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PlayerController->GetLocalPlayer());
        if (Subsystem)
        {
            Subsystem->AddMappingContext(InputMappingContext, 100);
        }
    }

    // 绑定输入动作
    UEnhancedInputComponent* EnhancedInputComponent = CastChecked<UEnhancedInputComponent>(PlayerController->InputComponent);
    if (EnhancedInputComponent)
    {
        if (IA_Capture)
        {
            EnhancedInputComponent->BindAction(IA_Capture, ETriggerEvent::Started, this, &UA_ViewFinder_Component::Capture_Photo);
        }
    }
}
```

这种设计允许玩家通过按键触发拍照功能，使系统可以自然地集成到游戏交互中。

## 总结

该拍照与物体投影系统是一个复杂而创新的游戏机制，结合了程序化网格、空间变换、射线检测和物理模拟等多种技术。系统通过采样物体形状，记录切片信息，然后在投影时重建物体，实现了一个独特的游戏玩法。

整个系统的工作原理是：相机通过射线检测和网格切割采样物体形状，存储物体的变换和切片信息，然后在投影时根据这些信息重建物体，包括应用所有切片操作和物理属性。这种方法允许玩家"复制"游戏世界中的物体，并在其他位置重建它们，创造了丰富的游戏可能性。