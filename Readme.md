# DazToHue Physics Tools

The following two nodes can be used to create UE PhysicsAssets using the DazToHue Houdini product. 

# DthPhysicsAsset

The workflow is centered around create templates that apply to one or more joints, defined by the `group_joints` field. Each folder and field in that template has a toggle, which if set, the values are written as point attribs on all joints matching the group glob. 

Multiple templates can write or overwrite attribs on the same joint. Templates are applied in sequence, so the last one takes precedent.  Sort of like CSS, you can use this to specify high level rules that effect many joints, and then make more specific rules to apply to subsets or even single joints. This allows for easier adjustment and tuning, as you usually only need to make a change in one place. 

Once the templates are all defined, there are buttons to export the bodies and the constraints. This creates a text string that UE accepts and copies it to the system clipboard. Switch to the UE PhysicsAssetEditor window for your SkelMesh and press `Ctrl-Shift-V` to paste the bodies or constraints. 

Bake times for computing convex colliders is about 30s on my machine, vs. ~4 min for the similar operation in UE. (TODO: numbers @ base resolution. Might need to throw in a polyreduce node for higher subD meshes.) Use `Iso` and `Exclude` when working on specific bodies if you don't want it to rebake all of them each when making changes. 

## Pre Setup

1. Follow the DazToHue workflow and import everything into Unreal.

2. Open the newly created PhysicsAsset in Unreal. Delete all the bodies. 

3. Add a sphere collider to the root bone, make it about 10cm. (You may have to right click in the Skeleton Tree and select `Show all bones` to see the root bone.) 

## Instructions

1. This node should follow the DazToHueSkeleton node, but the outputs **should not** be fed into the rest of the network, (however you can string multiple DthPhysicsAsset nodes together, useful to work on skeletal and PhatTissue bones separately.)

2. If using PhatTissue bones (i.e. breast/glute), you may want to modify the mesh with a `jointcapturepaint` node, making the pelvis, thighs, and spine_04 weights smaller so the colliders don't protrude past the PhatTissue colliders. We're not feeding this back into DazToHue, so don't worry about making it smooth. 

3. Enter the file address for `templates_skeletal.csv` next to and click `Import Setups CSV`. This is roughly tuned for the Genesis 9 Female base figure.

4. Check the display flag on the node and make sure the colliders look reasonable. 

5. Click `Export Both` in the controls section. This will copy to your system clipboard the data for your bodies and constraints. (These controls are duplicated at the bottom for convenience).

## Unreal Instructions

1. Back in the Unreal PhysicsAssetEditor, right click in the Skeleton Tree and click  `Paste all Constraints/Bodies`.  Test the constraints by simulating. 

1. If a constraint is behaving incorrectly, in the details panel trying reseting the Parent Rotation under Constraint Transforms. See note below on how to reflect the change in Houdini. 

1. As long as you aren't removing constraints, you can always export and paste changes you make in Houdini. 

1. To enable collision between the bodies, select all bodies and click `Enable Collision` on the tool bar. We'll then have to disable collision between certain bodies. (AFAICT there's no way to automate this without making a C++ UE plugin, which I likely will do)

1. Select all the bones in the left breast, clavicle_l, clavicle_pec_l, and spine 4 and 5. Then click `Disable Collision` on the tool bar, and deselect thebones. 

1. Do the same with right breast then the following groups (left and right separately):
  
  * glute, thigh, and pelvis
  * foot, heel, metatarsal
  * metacarpals
  * anything else that still looks jittery


## Body/Constraint Templates

`Collapse`: if checked, collapses the template other than the controls and group, but it will still be applied. 

`Iso`: if checked in any template, will only apply body templates with `Iso` checked.

`Exclude`: if checked, this template won't be applied. (Other templates affecting the group joints will still be applied.)

`group_joints`: group of joints this template applies to, using the Houdini glob syntax (e.g. to get all the toe bones, use "@name=*toe*")

## Template fields

Folders and fields in each template generally correspond roughly with `FSkeletalBodySetup`, `FConstraintInstance` and `FConstraintProfileProperties` Unreal structs. Fields that don't correspond to UE naming conventions (e.g. `weight_cutoff`) are specific to this HDA. 

### Convex bodies

`weight_cutoff`: How much a point on the mesh needs to be painted with a bone capture weight to be included in the convex hull for that bone. 0 to 1, higher numbers means less points will be included and smaller colliders. 

`overlap`: mainly used for bones with multiple colliders. Positive numbers grow the mesh point selection (bigger colliders) and negative shrink the selection. 

`copy_twist_bones`: recommended to be checked on the twist bones (thigh, upper and lower arm). This creates colliders on the twist bones, then cuts and pastes them to the parent bones with appropriate transformations. (I've not had any luck with separate colliders on twists that doesn't leave noodle arms.)

`Density`: doesn't get used in UE, but can be used for calculating drive strength. Higher density -> stronger drive strength where that body is the child body. 

`bExtraSpherd`: not implemented. Will add a sphere collider on limb joints to prevent gaps when the limb is moved from rest position.

**NOTE**: you can add a `Joint Capture Paint` node before the DthPhysicsAsset node to adjust the weights on certain bones. It's helpful to adjust spine_04 and pelvis if you're using the PhatTissue bones for breasts and/or glutes.  

### Constraint

***IMPORTANT***
`Set from UE Rotator`: This is used to correct alignment on joints that need it. In Unreal, if a constraint is not aligned, first try resetting the parent `Rotation` vector under `Constraint Transforms` in the `Details` tab. You may need to further adjust the constraint (you can alt-drag the rotation gizmo in the editor) and simulate to tune it.

Once the constraint is adjusted, right-click on the `Rotation` field and click copy. Paste this into the Houdini field and click the `Set from UE Rotator` button. This will populate the `AngularRotationOffset` field under `DefaultInstance`, ensure that field's toggle is checked. (This is a UE class member, but apparently unused in UE from what I gather. This HDA uses that value to calculate the final parent reference frame. It also sets `OrientationTarget` for the angular such that it aligns with the UE rest pose for that joint. ***WARNING*** edge cases may pop up here, I'm not satisfied with the correctness.) 

You will likely need to use this to correct joints whose axis of rotation differs from the x-vector of the joint itself. Certain joints such as the clavicle, clavicle_pec, and thigh may need a separate template for the right side if the standard mirroring fails. 

`constraint_type`:

* `Parent`: Joints in this group will have a constraint to their parent in the bone hierarchy.

* `Closest N as Child`\*: For each joint in the group, a constraint will be made to the N closest joints in the target group. Useful for constraining PhatTissue  bones to other PhatTissue bones

* `Closest N as Parent`\*: Same as above, but each point in `group_points` will be the parent. Useful for constraining PhatTissue bones to skeletal bones.

\* `Parent` constraint should be defined on the same template used to create the SkeletalBodySetup. The other two should be defined on separate templates after the body/parent joint template, and should not modify the SkeletalBodySetup. 

`force_mass_ratio`: Higher number -> higher linear and angular drive strength

`stiff_damp_ratio`: Higher number -> lower linear/angular damping strength.




#### DefaultInstance



***

# DthPhatTissue

Creates additional DazToHue bones for the breast and glutes, allowing for more realistic PhysicsAssets in unreal. The node goes after the DazToHueSkeleton node, and you should string two nodes together if you want bones for both breasts and glutes. 

The node will 

## Pre setup

In your DazToHueSkeleton node, you should create and position the breast and glute bones you want. I used 4 glute bones and 5 breast bones, and may have made assumptions that require that number. 

You will also need to paint `breast_density` and `glute_density` weights on the mesh using an `attribpaint` node. Only paint the left side, it will be mirrored automatically. Bones will be projected onto the mesh based on the weight, with higher weighted regions having more bones. This doesn't need to be pretty, and by setting the display flag on the DthPhatBones node, you can paint while seeing the final result. 

**WARNING** This node uses a `scatter` node to place the leaf bones on the mesh, which means it's not deterministic. If you change anything leading up to the node, it may alter the skeleton, and you will have to re-export from the DazToHueExport node and reimport to UE. 

## Node Parms. 

Setting the `Phat tissue type` should fill in reasonable defaults to most of the other parms.(Stomach not implemented.) 

`Number of core bones`: **Must** match the number from DazToHueSkeleton node.

`Distance core->spoke`, `Z-offset core->spoke` and `Leaf distance from mesh` can be adjusted 
