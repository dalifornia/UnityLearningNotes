# Red Dead Redemption2_Dead Eye
*Shawn, 2022-6-17*
* 该md主要记录了在Unity里实现荒野大镖客2中死神之眼技能的主要步骤，用做学习记录。
------------------------
原项目和作者信息
* Name: Mix And Jam
* GitHub: https://github.com/mixandjam/RDR-DeadEye/blob/master
* Youtube: https://youtu.be/jPnMVeWnZLc
------------------------
相关资源：
* 基于Cinemachine的第三人称控制器: https://youtu.be/J1GgvDfmIo0
* 角色模型和动画下载：Mixamo
* 枪械模型下载：https://sketchfab.com/3d-models/flintlock-rifle-6605ddbf0dd2411690ac58e46213fa80


### 1.导入模型与动画
* Mixamo Character:
    * 建议选择YBot作为角色模型，并下载YBot配套的动画，以避免骨骼绑定不匹配的问题
* Mixamo Animation:
    * Rifle Idle
    * Walk With Rifle_1
    * Walk With Rifle_2

### 2. 在Unity中导入动画，Avtar选择Humanmoid，Create from this model
* 在Rig中点击configuration按钮，调整骨骼角度，保证动画正确
* 设置Movement Blend Tree
* 若角色行走不是直线，则修改RootTransform的朝向，将BodyOrientation 改为 Original

### 3. Cinemachine -> Create FreeLook Camera
* 调整 Top Middle Bottom Rig的角度，确保镜头角度限制
* 设置Offset，调整角色在镜头中的位置

### 4.绑定脚本MovementInput
* 基于FileStorm制作的第三人称控制器，微调；
* 添加重力:

    ```c#
    if (controller.isGrounded) {
	    verticalVel -= 0.5f * Time.deltaTime;
    }
    moveVector = new Vector3 (0, verticalVel, 0);
    moveVector *= Time.deltaTime;
    controller.Move (moveVector);
    ```

### 5. 下载Rifle model并绑定到角色手部
* Sketchfab-> Model name:Flintlock Rifle-> Author:Robert Berrier
* 使用unity默认的IK，根据IK Goal绑定武器，Animation->Layer->IK Pass->Script->OnAnimatorIK()

### 6. 瞄准的设计与实现
* 主要思想：修改相机的FOV和Middle Rig Offset
* 注意事项：因为模型和动画不匹配，即Avatar不一致，微调角色的骨骼绑定只能略微修复动画的走样，且Unity的默认IK绑定也不能完全解决武器与手部的贴合问题，因此瞄准的时候枪支可能会与手部偏离严重，因此基于DoTween实现了武器在不同状态下的Transform位移。
    ```csharp
    private void WeaponPosition()
    {
        bool state = input.Speed > 0;

        Vector3 pos = state ? gunAimPos : gunIdlePos;
        Vector3 rot = state ? gunAimRot : gunIdleRot;
        gun.DOLocalMove(pos, .3f);
        gun.DOLocalRotate(rot, .3f);
    }
    ```
    `WeaponPostion()`方法用于修正武器的Transform，当角色移动时，进入瞄准姿势，因此需要将武器的Transform修改为瞄准姿势下贴合手部的transform。而DoTween提供的方法`DOLocalMove()`,`DOLocalRotate()`为修改Transform提供了顺滑的移动过程。

### 7.利用射线碰撞来检测是否瞄准到Enemy对象
    ```csharp
    RaycastHit hit;
    Physics.Raycast(Camera.main.transform.position, Camera.main.transform.forward, out hit);
    ```
    此处的hit对象即包含着射线碰撞点的信息，判断碰撞物体的Tag即可判断出是否瞄准到了Enemy对象。若是，则随后将其Transform加入到target列表中。
    ```csharp
    if (!targets.Contains(hit.transform) && !hit.transform.GetComponentInParent<EnemyScript>().aimed)
    {
        // Add hit transform
        hit.transform.GetComponentInParent<EnemyScript>().aimed = true;
        targets.Add(hit.transform);
        // Add Cross transform
        Vector3 convertedPos = Camera.main.WorldToScreenPoint(hit.transform.position);
        GameObject cross = Instantiate(aimPrefab, canvas);
        cross.transform.position = convertedPos;
        crossList.Add(cross.transform);
    }
    ```

### 8.瞄准后获取目标，加入Sequence中，并添加回调事件
    ```csharp
    Sequence s = DOTween.Sequence();    // Create DoTween.Sequence
    for (int i = 0; i < targets.Count; i++)
    {
        // Add Animation
        s.Append(transform.DOLookAt(targets[i].GetComponentInParent<EnemyScript>().transform.position, .05f).SetUpdate(true));
        // Add callback functions
        s.AppendCallback(() => anim.SetTrigger("fire"));    // Trigger Fire Animation
        int x = i;
        s.AppendInterval(.05f);
        s.AppendCallback(()=>FirePolish());                 // Play Fire Particle
        s.AppendCallback(() => targets[x].GetComponentInParent<EnemyScript>().Ragdoll(true, targets[x]));   // Activate enemy's Ragdoll system
        s.AppendCallback(() => crossList[x].GetComponent<Image>().color = Color.clear);                     // Let cross disappear
        s.AppendInterval(.35f);
    }
    s.AppendCallback(() => Aim(false));     // Set aim state false
    s.AppendCallback(() =>DeadEye(false));  // Set deadeye state false
    ```

### 9.Enemy击杀效果
* 根据原作者Mix And Jam的思路，为Enemy对象的四肢、躯干、头部添加RigidBody组件，在脚本EnemyScript.cs中实现布娃娃系统。
* 该步骤的实现逻辑比较简单：
    ```csharp
    foreach (Rigidbody rb in rbs)
    {
        rb.isKinematic = !state;
    }
    if(state == true)
    {
        point.GetComponent<Rigidbody>().AddForce(shooter.transform.forward * 30, ForceMode.Impulse);
    }
    ```
    若该Enemy被DeadEye标记，则取消其刚体的动力学属性，并在标记点上施加一个方向为transform.forward的力，以表现其被击中后的布娃娃效果。
* PS：本人之前没看到有EnemyScript这个脚本，还以为该步骤里面确实有子弹一类的对象生成，一直在找子弹与Enemy碰撞时命中逻辑如何实现，白费了很多时间。转念一想，确实也是，射击类游戏里命中的关键要素在于命中的位置，没必要一定要实现子弹飞行、碰撞的完整过程。


### 10.后期润色
* 该部分由于本人初学Unity不到半年，期间还在完成与Unity无关的毕业设计，所以还没来得及完全学习粒子系统、光源、渲染一类的知识。
* 该部分建议照搬原项目中的组件
* Particle System
* Post-Processing

