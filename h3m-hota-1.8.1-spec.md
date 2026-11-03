# Heroes III: Horn of the Abyss 1.8.1 map file format

This specification describes HotA map revision **10**. An `.h3m` file stores the map payload in a gzip stream. Decompress it before reading the fields below. Offsets refer to the decompressed stream, starting at zero.

## Notation

- Integers are little-endian. The number in `uint8`, `int16`, etc. is the width in bits. Signed integers use two's complement. Fields are packed in the order shown, with no alignment padding.
- `[Type] field` means one inline record of the named type when the type is capitalized. `field[n]` means exactly `n` elements. A zero-element array consumes zero bytes. A bracketed argument after a string field gives its byte length. Strings have a preceding `uint32` byte count, followed by those raw text bytes. Zero length consumes no text bytes. Treat text as raw bytes at the file-format layer.
- A condition or loop controls field presence. Record parameters in parentheses supply context and consume no bytes. `let` is a computed value with no serialized bytes. `field : n` denotes a packed bitfield of width `n`, starting at the least significant bit. Byte arrays used as ID masks address bit `i % 8` in byte `i / 8` using integer division. All divisions in the layouts are integer divisions.
- Flags represented as one-byte Booleans use 0 for false and 1 for true in ordinary files. Values named unknown, skipped, garbage or reserved are still part of the layout. Preserve their bytes when editing an existing map.

## Payload order

1. [General information](#1-general-information)
2. [Players](#2-players)
3. [Victory and loss conditions](#3-victory-and-loss-conditions)
4. [Teams](#4-teams)
5. [Starting heroes](#5-starting-heroes)
6. [Settings before extended events](#6-settings-before-extended-events)
7. [Extended events](#7-extended-events)
8. [Settings after extended events](#8-settings-after-extended-events)
9. [Rumors](#9-rumors)
10. [Hero templates](#10-hero-templates)
11. [Terrain](#11-terrain)
12. [Object definitions](#12-object-definitions)
13. [Placed objects](#13-placed-objects)
14. [Global events](#14-global-events)
15. [Trailer](#15-trailer)

[Numeric identifiers](#numeric-identifiers)

## 1. General information

The first five values for a map saved by HotA 1.8.1 are normally 0x20, 10, 1, 8, 1. Revision 10 selects the layout described here. The release fields and version-lock flag are separate from the layout revision.

### GeneralSpecs

```
[uint32] map_format;
[uint32] hota_map_version;
[uint32] hota_major_version;
[uint32] hota_minor_version;
[uint32] hota_patch_version;
[uint8] is_mirror;
[uint8] is_arena;
[uint32] terrain_type_count;
[uint32] town_type_count;
[DifficultyBits] allowed_difficulties;
[uint8] can_hire_defeated_heroes;
[uint8] is_hota_version_locked;

[uint32] variable_count;
[H3MVariable] variables[variable_count];

[uint8] has_hero;
[uint32] map_size;
[uint8] has_underground;
[uint32] map_name_length;
[string] map_name[map_name_length];
[uint32] description_length;
[string] description[description_length];
[uint8] difficulty;
[uint8] level_cap;
```

### Coords

```
[uint8] x;
[uint8] y;
[uint8] z;
```

### DifficultyBits

```
[uint8] easy       : 1;
[uint8] normal     : 1;
[uint8] hard       : 1;
[uint8] expert     : 1;
[uint8] impossible : 1;
[uint8] reserved   : 3;
```

### TownBits

```
[uint8] castle     : 1;
[uint8] rampart    : 1;
[uint8] tower      : 1;
[uint8] inferno    : 1;
[uint8] necropolis : 1;
[uint8] dungeon    : 1;
[uint8] stronghold : 1;
[uint8] fortress   : 1;
[uint8] conflux    : 1;
[uint8] cove       : 1;
[uint8] factory    : 1;
[uint8] bulwark    : 1;
[uint8] reserved   : 4;
```

### H3MVariable

```
[uint32] name_length;
[string] name[name_length];
[int32] initial_value;
[uint8] value_mode;
```

## 2. Players

Player slots, player masks, team assignments and hiring flags follow the order in the player color table.

### PlayerSpecs

```
[PlayerSpec] players[8];
```

### AvailableHero

```
[uint8] id;
[uint32] custom_name_length;
[string] custom_name[custom_name_length];
```

### PlaceholderHero

```
[uint8] raw[5];
```

### PlayerSpec

```
[uint8] human_playable;
[uint8] ai_playable;
[uint8] ai_behavior;
[uint8] allowed_towns_customized;
[TownBits] allowed_towns;
[uint8] has_random_town;
[uint8] has_main_town;
if (has_main_town)
{
    [uint8] generate_hero;
    [uint8] town_type;
    [Coords] town_coords;
}

[uint8] has_random_hero;
[uint8] starting_hero_id;
if (starting_hero_id != 255)
{
    [uint8] starting_hero_face;
    [uint32] starting_hero_name_length;
    [string] starting_hero_name[starting_hero_name_length];
    [uint8] garbage_byte;
    [uint32] available_hero_count;
    [AvailableHero] available_heroes[available_hero_count];
}
else
{
    [uint8] skipped_byte;
    [uint32] placeholder_hero_count;
    [PlaceholderHero] placeholder_heroes[placeholder_hero_count];
}
```

## 3. Victory and loss conditions

### VictoryLossConditions

```
[uint8] victory_condition;
if (victory_condition != VC_None)
{
    [uint8] allow_normal_win;
    [uint8] allow_ai_special_win;

    if (victory_condition == VC_AcquireArtifact)
        [uint16] artifact_id;
    else if (victory_condition == VC_AccumulateCreatures)
    {
        [uint16] creature_id;
        [uint32] creature_amount;
    }
    else if (victory_condition == VC_AccumulateResources)
    {
        [uint8] resource_id;
        [uint32] resource_amount;
    }
    else if (victory_condition == VC_UpgradeTown)
    {
        [Coords] town_coords;
        [uint8] town_hall_level;
        [uint8] fort_level;
    }
    else if (victory_condition == VC_BuildTheGrail ||
             victory_condition == VC_DefeatHero ||
             victory_condition == VC_CaptureTown ||
             victory_condition == VC_DefeatMonster)
        [Coords] objective_coords;
    else if (victory_condition == VC_TransportArtifact)
    {
        [uint8] artifact_id;
        [Coords] destination_coords;
    }
    else if (victory_condition == VC_Survive)
        [uint32] days;
}

[uint8] loss_condition;
if (loss_condition == LC_TimeExpires)
    [uint16] loss_timer;
else if (loss_condition != LC_None)
    [Coords] loss_coords;
```

## 4. Teams

### TeamSpecs

```
[uint8] amount_of_teams;
if (amount_of_teams != 0)
    [uint8] player_team[8];
```

## 5. Starting heroes

### StartingHeroes

```
[uint32] total_heroes;
[uint8] hero_flags[27];

[uint32] placeholder_hero_count;
[uint8] placeholder_heroes[placeholder_hero_count];

[uint8] custom_hero_count;
[StartingHeroCustom] custom_heroes[custom_hero_count];

[uint8] unhandled_bytes[31];
```

### StartingHeroCustom

```
[uint8] id;
[uint8] face;
[uint32] custom_name_length;
[string] custom_name[custom_name_length];
[uint8] may_be_hired_by;
```

## 6. Settings before extended events

### Settings1

```
[uint32] allow_plague;
[uint32] combo_artifact_count;
let combo_artifact_bytes = (combo_artifact_count + 7) / 8;
[uint8] combo_artifacts[combo_artifact_bytes];
[uint32] combat_round_limit;

[uint8] forbid_hiring[8];
```

## 7. Extended events

Expression, condition and action records are recursive.

### ExtendedEvents

```
[uint8] has_extended_events;

if (has_extended_events)
{
    [ExtendedEventSet] hero_events;
    [ExtendedEventSet] player_events;
    [ExtendedEventSet] town_events;
    [ExtendedEventSet] quest_events;

    [ExtendedEventCounters] counters;

    [uint32] variable_count;
    [ExtendedEventVariable] variables[variable_count];

    [ExtendedEventIdList] hero_ids;
    [ExtendedEventIdList] player_ids;
    [ExtendedEventIdList] town_ids;
    [ExtendedEventIdList] quest_ids;
    [ExtendedEventIdList] variable_ids;
}
```

### ExtendedEventVariable

```
[uint32] id;
[uint32] name_length;
[string] name[name_length];
[uint8] save_in_campaign;
[uint8] value_mode;
if (value_mode == VAR_InitialValue)
    [int32] value;
```

### ExtendedEventExpression

The resource expression uses a player color ID or 255 for the current player.

```
[uint8] item_count_or_flag;
[uint32] expression_type;

if (expression_type == EE_EXPR_IntegerValue)
    [int32] integer_value;
else if (expression_type == EE_EXPR_VariableValue)
    [uint32] variable_id;
else if (expression_type == EE_EXPR_Resource)
{
    [uint8] player;
    [uint32] resource;
}
else if (expression_type == EE_EXPR_CreatureCount)
    [uint32] creature_id;
else if (expression_type == EE_EXPR_DifficultyLevel)
    [uint32] difficulty_level;
else if (expression_type == EE_EXPR_PrimarySkill)
    [uint32] primary_skill;
else if (expression_type == EE_EXPR_ArtifactCount)
{
    [uint32] artifact_id;
    if (artifact_id == ARTIFACT_SpellScroll)
        [uint32] spell_id;
    else
        [uint32] artifact_spell_id_none;
}
else if (expression_type == EE_EXPR_CurrentDifficultyLevel ||
         expression_type == EE_EXPR_CurrentDate ||
         expression_type == EE_EXPR_CurrentHeroExperience ||
         expression_type == EE_EXPR_CurrentHeroLevel)
{
}
else if (expression_type == EE_EXPR_InvertSign)
    [ExtendedEventExpression] value_expression;
else if (expression_type == EE_EXPR_Add ||
         expression_type == EE_EXPR_Subtract ||
         expression_type == EE_EXPR_Multiply ||
         expression_type == EE_EXPR_Divide ||
         expression_type == EE_EXPR_Remainder)
{
    [ExtendedEventExpression] left_expression;
    [ExtendedEventExpression] right_expression;
}
else if (expression_type == EE_EXPR_RandomNumber)
{
    [uint8] min_mode;
    if (min_mode == EE_EXPR_OPERAND_Integer)
        [int32] min_integer_value;
    else if (min_mode == EE_EXPR_OPERAND_Expression)
        [ExtendedEventExpression] min_expression;

    [uint8] max_mode;
    if (max_mode == EE_EXPR_OPERAND_Integer)
        [int32] max_integer_value;
    else if (max_mode == EE_EXPR_OPERAND_Expression)
        [ExtendedEventExpression] max_expression;
}
```

### ExtendedEventConditionValue

```
[uint8] mode;
if (mode == EE_EXPR_OPERAND_Integer)
    [int32] integer_value;
else if (mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] expression;
```

### ExtendedEventConditionNode

Player fields use the shared color IDs. Player Is Human and Player Starting Town also accept -1 for the current player. Monster Defeated, Hero Defeated and Town Controlled also accept -1 for the current player or -2 for any player. Player Starting Town uses IDs 0–11 from the town type table.

```
[uint32] condition_type;

if (condition_type == EE_COND_Boolean)
    [uint8] boolean_value;
else if (condition_type == EE_COND_HeroHasArtifact)
{
    [uint32] artifact_id;
    if (artifact_id == ARTIFACT_SpellScroll)
        [uint32] spell_id;
    else
        [uint32] artifact_spell_id_none;
}
else if (condition_type == EE_COND_MonsterDefeatedByPlayer)
{
    [int32] player;
    [uint32] monster_uid;
}
else if (condition_type == EE_COND_HeroDefeatedByPlayer)
{
    [int32] player;
    [uint32] hero_uid;
}
else if (condition_type == EE_COND_TownControlledByPlayer)
{
    [int32] player;
    [uint32] town_uid;
}
else if (condition_type == EE_COND_CurrentHeroId)
    [uint32] hero_name;
else if (condition_type == EE_COND_CurrentHeroPlayer ||
         condition_type == EE_COND_PlayerDefeated)
    [uint32] player;
else if (condition_type == EE_COND_PlayerIsHuman)
    [int32] player;
else if (condition_type == EE_COND_PlayerStartingTown)
{
    [int32] player;
    [uint32] town_type;
}
else if (condition_type == EE_COND_HeroAffiliation)
{
    [uint32] hero_name;
    [int32] location;
}
else if (condition_type == EE_COND_HeroClass)
    [uint32] hero_class;
else if (condition_type == EE_COND_HeroSecondarySkill)
{
    [uint32] secondary_skill;
    [uint32] secondary_skill_level;
}
else if (condition_type == EE_COND_LessThan ||
         condition_type == EE_COND_GreaterThan ||
         condition_type == EE_COND_Equal ||
         condition_type == EE_COND_GreaterThanOrEqual ||
         condition_type == EE_COND_LessThanOrEqual ||
         condition_type == EE_COND_NotEqual)
{
    [ExtendedEventConditionValue] left_value;
    [ExtendedEventConditionValue] right_value;
}
else if (condition_type == EE_COND_Not)
    [ExtendedEventCondition] nested_condition;
else if (condition_type == EE_COND_And || condition_type == EE_COND_Or)
{
    [uint32] condition_count;
    [ExtendedEventConditionNode] child_conditions[condition_count];
}
```

### ExtendedEventCondition

```
[uint8] item_count_or_flag;
[ExtendedEventConditionNode] node;
```

### ExtendedEventMessageImage

```
[uint32] image_type;
if (image_type == EE_MSG_IMAGE_ResourcesPlus ||
    image_type == EE_MSG_IMAGE_ResourcesMinus ||
    image_type == EE_MSG_IMAGE_ResourcesPerDay ||
    image_type == EE_MSG_IMAGE_ResourcesSimple)
{
    [uint32] resource;
    [uint8] amount_mode;
    if (amount_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] amount;
    else if (amount_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_Artifact)
{
    [uint32] artifact_id;
    [uint8] hidden_value_mode;
    if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] hidden_value;
    else if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_Spell ||
         image_type == EE_MSG_IMAGE_SpellScroll)
{
    [uint32] spell_id;
    [uint8] hidden_value_mode;
    if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] hidden_value;
    else if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_PlayerFlag)
{
    [uint32] player;
    [uint8] hidden_value_mode;
    if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] hidden_value;
    else if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_Luck ||
         image_type == EE_MSG_IMAGE_Morale)
{
    [int32] value;
    [uint8] hidden_value_mode;
    if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] hidden_value;
    else if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_Experience ||
         image_type == EE_MSG_IMAGE_GoldSmallPlus ||
         image_type == EE_MSG_IMAGE_GoldSmallMinus ||
         image_type == EE_MSG_IMAGE_GoldSmall ||
         image_type == EE_MSG_IMAGE_GoldSmallPerDay)
{
    [int32] subtype_placeholder;
    [uint8] amount_mode;
    if (amount_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] amount;
    else if (amount_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_LevelPlusOne)
{
    [int32] subtype_placeholder;
    [uint8] hidden_value_mode;
    if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] hidden_value;
    else if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_SecondarySkill)
{
    [uint32] secondary_skill;
    [uint8] level_mode;
    if (level_mode == EE_MSG_IMAGE_VALUE_Integer)
        [uint32] secondary_skill_level;
    else if (level_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_Creatures ||
         image_type == EE_MSG_IMAGE_CreaturesSimple)
{
    [uint32] creature_id;
    [uint8] amount_mode;
    if (amount_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] amount;
    else if (amount_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_PrimarySkill ||
         image_type == EE_MSG_IMAGE_PrimarySkillPlus)
{
    [uint32] primary_skill;
    [uint8] amount_mode;
    if (amount_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] amount;
    else if (amount_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type == EE_MSG_IMAGE_SpellPoints ||
         image_type == EE_MSG_IMAGE_MovementPoints)
{
    [int32] direction;
    [uint8] amount_mode;
    if (amount_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] amount;
    else if (amount_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
else if (image_type >= EE_MSG_IMAGE_BuildingCastle &&
         image_type <= EE_MSG_IMAGE_BuildingBulwark)
{
    [uint32] building;
    [uint8] hidden_value_mode;
    if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Integer)
        [int32] hidden_value;
    else if (hidden_value_mode == EE_MSG_IMAGE_VALUE_Variable)
        [uint32] variable_id;
}
```

### ExtendedEventArtifactRewardAction

```
[uint8] operation;
[uint32] artifact_id;
if (artifact_id == ARTIFACT_SpellScroll)
    [uint32] spell_id;
else
    [uint32] artifact_spell_id_none;
[uint8] show_with_message;
```

### ExtendedEventCreaturesRewardAction

```
[uint8] operation;
[uint32] creature_id;
[uint8] quantity_mode;
if (quantity_mode == EE_EXPR_OPERAND_Integer)
    [int32] quantity;
else if (quantity_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] quantity_expression;
[uint8] show_with_message;
```

### ExtendedEventResourceAmount

```
[uint8] mode;
if (mode == EE_EXPR_OPERAND_Integer)
    [int32] amount;
else if (mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] expression;
```

### ExtendedEventResourcesRewardAction

```
[uint8] operation;
[ExtendedEventResourceAmount] wood;
[ExtendedEventResourceAmount] mercury;
[ExtendedEventResourceAmount] ore;
[ExtendedEventResourceAmount] sulfur;
[ExtendedEventResourceAmount] crystal;
[ExtendedEventResourceAmount] gems;
[ExtendedEventResourceAmount] gold;
[uint8] show_with_message;
```

### ExtendedEventSpellRewardAction

```
[uint32] spell_id;
[uint8] show_with_message;
```

### ExtendedEventExperienceRewardAction

```
[uint8] amount_mode;
if (amount_mode == EE_EXPR_OPERAND_Integer)
    [int32] amount;
else if (amount_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] expression;
[uint8] show_with_message;
```

### ExtendedEventSpellPointsRewardAction

```
[uint8] amount_mode;
if (amount_mode == EE_EXPR_OPERAND_Integer)
    [int32] amount;
else if (amount_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] expression;
[uint32] operation;
[uint8] show_with_message;
```

### ExtendedEventMovementPointsRewardAction

```
[uint8] amount_mode;
if (amount_mode == EE_EXPR_OPERAND_Integer)
    [int32] amount;
else if (amount_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] expression;
[uint32] operation;
[uint8] show_with_message;
```

### ExtendedEventPrimarySkillsRewardAction

```
[uint8] amount_mode;
if (amount_mode == EE_EXPR_OPERAND_Integer)
    [int32] amount;
else if (amount_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] expression;
[uint32] primary_skill;
[uint8] show_with_message;
```

### ExtendedEventSecondarySkillsRewardAction

```
[uint32] secondary_skill_level;
[uint32] secondary_skill;
[uint8] show_with_message;
```

### ExtendedEventSignedAmountRewardAction

```
[int32] amount;
[uint8] show_with_message;
```

### ExtendedEventAddCreaturesToHireAction

```
[uint32] creature_level; // Extended event town creature levels
[uint8] quantity_mode;
if (quantity_mode == EE_EXPR_OPERAND_Integer)
    [int32] quantity;
else if (quantity_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] quantity_expression;
[uint32] unused_creature_id;
[uint8] show_with_message;
```

### ExtendedEventBuildTownStructureAction

The town type uses IDs 0–11 from the town type table or 65535 for a common building.

```
[uint32] building;
[uint32] town_type;
[uint8] show_with_message;
```

### ExtendedEventQuestLogImage

```
[uint32] image_type;

if (image_type == EE_QUEST_LOG_IMAGE_Artifact)
    [uint32] artifact_id;
else if (image_type == EE_QUEST_LOG_IMAGE_SpellScroll)
    [uint32] spell_id;
else if (image_type == EE_QUEST_LOG_IMAGE_Creatures)
    [uint32] creature_id;
else if (image_type == EE_QUEST_LOG_IMAGE_Resources)
    [uint32] resource;
else if (image_type == EE_QUEST_LOG_IMAGE_PrimarySkill)
    [uint32] primary_skill;
else if (image_type == EE_QUEST_LOG_IMAGE_SecondarySkill)
    [uint32] secondary_skill;
else if (image_type == EE_QUEST_LOG_IMAGE_Level)
    [int32] level_placeholder;
else if (image_type == EE_QUEST_LOG_IMAGE_PlayerFlag)
    [uint32] player;
else if (image_type == EE_QUEST_LOG_IMAGE_Hero)
    [uint32] hero_name;
else if (image_type == EE_QUEST_LOG_IMAGE_HeroOnMap)
    [uint32] hero_uid;
else if (image_type == EE_QUEST_LOG_IMAGE_MonsterOnMap)
    [uint32] monster_uid;

[uint8] value_mode;
if (value_mode == EE_MSG_IMAGE_VALUE_Integer)
    [int32] value;
else if (value_mode == EE_MSG_IMAGE_VALUE_Variable)
    [uint32] variable_id;
```

### ExtendedEventSetQuestHintAndLogEntryAction

```
[uint32] quest_hint_length;
[string] quest_hint_text[quest_hint_length];

[uint32] quest_log_image_count;
[ExtendedEventQuestLogImage] quest_log_images[quest_log_image_count];

[uint8] show_quest_in_log;
```

### ExtendedEventCombatStack

```
[uint8] quantity_mode;
if (quantity_mode == EE_EXPR_OPERAND_Integer)
    [int32] quantity;
else if (quantity_mode == EE_EXPR_OPERAND_Expression)
    [ExtendedEventExpression] quantity_expression;
[int32] creature_id;
```

### ExtendedEventCombatAction

```
[ExtendedEventCombatStack] slots[7];
```

### ExtendedEventWarMachineRewardAction

```
[uint8] operation;
[uint32] war_machine_artifact_id;
[uint32] hidden_extra;
[uint8] show_with_message;
```

### ExtendedEventSpellbookRewardAction

```
[uint8] operation;
[uint32] hidden_extra_1;
[uint32] hidden_extra_2;
[uint8] show_with_message;
```

### ExtendedEventModifyVariableAction

```
[uint32] variable_id;
[uint8] operation;
[ExtendedEventExpression] expression;
```

### ExtendedEventRewardEntry

Reward entries use the subset of action IDs shown below.

```
[uint32] reward_type;

if (reward_type == EE_ACTION_ModifyVariable)
    [ExtendedEventModifyVariableAction] modify_variable;
else if (reward_type == EE_ACTION_ResourcesReward)
    [ExtendedEventResourcesRewardAction] resources_reward;
else if (reward_type == EE_ACTION_CreaturesReward)
    [ExtendedEventCreaturesRewardAction] creatures_reward;
else if (reward_type == EE_ACTION_ArtifactReward)
    [ExtendedEventArtifactRewardAction] artifact_reward;
else if (reward_type == EE_ACTION_BuildTownStructure)
    [ExtendedEventBuildTownStructureAction] build_town_structure;
else if (reward_type == EE_ACTION_AddCreaturesToHire)
    [ExtendedEventAddCreaturesToHireAction] add_creatures_to_hire;
else if (reward_type == EE_ACTION_SpellReward)
    [ExtendedEventSpellRewardAction] spell_reward;
else if (reward_type == EE_ACTION_ExperienceReward)
    [ExtendedEventExperienceRewardAction] experience_reward;
else if (reward_type == EE_ACTION_SpellPointsReward)
    [ExtendedEventSpellPointsRewardAction] spell_points_reward;
else if (reward_type == EE_ACTION_MovementPointsReward)
    [ExtendedEventMovementPointsRewardAction] movement_points_reward;
else if (reward_type == EE_ACTION_PrimarySkillsReward)
    [ExtendedEventPrimarySkillsRewardAction] primary_skills_reward;
else if (reward_type == EE_ACTION_SecondarySkillsReward)
    [ExtendedEventSecondarySkillsRewardAction] secondary_skills_reward;
else if (reward_type == EE_ACTION_LuckReward)
    [ExtendedEventSignedAmountRewardAction] luck_reward;
else if (reward_type == EE_ACTION_MoraleReward)
    [ExtendedEventSignedAmountRewardAction] morale_reward;
else if (reward_type == EE_ACTION_WarMachineReward)
    [ExtendedEventWarMachineRewardAction] war_machine_reward;
else if (reward_type == EE_ACTION_SpellbookReward)
    [ExtendedEventSpellbookRewardAction] spellbook_reward;
else if (reward_type == EE_ACTION_RemoveCurrentObjectFinishQuest ||
         reward_type == EE_ACTION_DisableEvent)
{

}
```

### ExtendedEventRewardList

```
[uint8] rewards_marker;
[uint32] rewards_unknown;
[uint32] reward_count;
[ExtendedEventRewardEntry] rewards[reward_count];
```

### ExtendedEventAction

```
[uint32] action_id;

if (action_id == EE_ACTION_SetVariableBasedOnCondition)
{
    [uint32] variable_id;
    [ExtendedEventCondition] condition;
    [ExtendedEventConditionValue] true_value;
    [ExtendedEventConditionValue] false_value;
}
else if (action_id == EE_ACTION_ModifyVariable)
{
    [uint32] variable_id;
    [uint8] operation;
    [ExtendedEventExpression] expression;
}
else if (action_id == EE_ACTION_ResourcesReward)
    [ExtendedEventResourcesRewardAction] resources_reward;
else if (action_id == EE_ACTION_MessageWithReward)
{
    [uint32] message_length;
    [string] message_text[message_length];
    [ExtendedEventRewardList] reward_list;
}
else if (action_id == EE_ACTION_QuestAction)
{
    [ExtendedEventCondition] condition;

    [uint32] proposal_message_length;
    [string] proposal_message_text[proposal_message_length];

    [uint32] progress_message_length;
    [string] progress_message_text[progress_message_length];

    [uint32] completion_message_length;
    [string] completion_message_text[completion_message_length];

    [uint32] hint_message_length;
    [string] hint_message_text[hint_message_length];

    [ExtendedEventActionList] completion_actions;
    [uint8] set_quest_log_entry;
}
else if (action_id == EE_ACTION_CreaturesReward)
    [ExtendedEventCreaturesRewardAction] creatures_reward;
else if (action_id == EE_ACTION_ArtifactReward)
    [ExtendedEventArtifactRewardAction] artifact_reward;
else if (action_id == EE_ACTION_WarMachineReward)
    [ExtendedEventWarMachineRewardAction] war_machine_reward;
else if (action_id == EE_ACTION_SpellbookReward)
    [ExtendedEventSpellbookRewardAction] spellbook_reward;
else if (action_id == EE_ACTION_Question)
{
    [uint8] image_mode;
    [uint32] question_length;
    [string] question_text[question_length];
    [ExtendedEventActionList] yes_actions;
    [ExtendedEventActionList] no_actions;
    if (image_mode == EE_QUESTION_IMAGES_SelectionCanExit)
        [ExtendedEventActionList] cancel_actions;

    if (image_mode == EE_QUESTION_IMAGES_NoImages ||
        image_mode == EE_QUESTION_IMAGES_SpecifyImages)
    {
        [uint32] specified_image_count;
        [ExtendedEventMessageImage] specified_images[specified_image_count];
    }
    else if (image_mode == EE_QUESTION_IMAGES_SelectionCantExit ||
             image_mode == EE_QUESTION_IMAGES_SelectionCanExit)
    {
        [ExtendedEventMessageImage] first_choice_image;
        [ExtendedEventMessageImage] second_choice_image;
        [uint8] show_or_text;
        [uint32] stale_specified_image_count;
        [ExtendedEventMessageImage] stale_specified_images[stale_specified_image_count];
    }
}
else if (action_id == EE_ACTION_AddCreaturesToHire)
    [ExtendedEventAddCreaturesToHireAction] add_creatures_to_hire;
else if (action_id == EE_ACTION_BuildTownStructure)
    [ExtendedEventBuildTownStructureAction] build_town_structure;
else if (action_id == EE_ACTION_SetQuestHintAndLogEntry)
    [ExtendedEventSetQuestHintAndLogEntryAction] set_quest_hint_and_log_entry;
else if (action_id == EE_ACTION_SpellReward)
    [ExtendedEventSpellRewardAction] spell_reward;
else if (action_id == EE_ACTION_ExperienceReward)
    [ExtendedEventExperienceRewardAction] experience_reward;
else if (action_id == EE_ACTION_SpellPointsReward)
    [ExtendedEventSpellPointsRewardAction] spell_points_reward;
else if (action_id == EE_ACTION_MovementPointsReward)
    [ExtendedEventMovementPointsRewardAction] movement_points_reward;
else if (action_id == EE_ACTION_PrimarySkillsReward)
    [ExtendedEventPrimarySkillsRewardAction] primary_skills_reward;
else if (action_id == EE_ACTION_SecondarySkillsReward)
    [ExtendedEventSecondarySkillsRewardAction] secondary_skills_reward;
else if (action_id == EE_ACTION_LuckReward)
    [ExtendedEventSignedAmountRewardAction] luck_reward;
else if (action_id == EE_ACTION_MoraleReward)
    [ExtendedEventSignedAmountRewardAction] morale_reward;
else if (action_id == EE_ACTION_Combat)
    [ExtendedEventCombatAction] combat;
else if (action_id == EE_ACTION_ExecuteEvent)
{
    [uint32] event_type;
    [uint32] event_id;
}
else if (action_id == EE_ACTION_ShowMessage)
{
    [uint32] message_length;
    [string] message_text[message_length];
    [uint32] image_count;
    [ExtendedEventMessageImage] images[image_count];
}
else if (action_id == EE_ACTION_DisableEvent ||
         action_id == EE_ACTION_RemoveCurrentObjectFinishQuest)
{
    // No payload.
}
else if (action_id == EE_ACTION_Cycle)
{
    [ExtendedEventActionList] sequence_actions;
    [ExtendedEventConditionValue] first_value;
    [ExtendedEventConditionValue] second_value;
    [uint32] variable_id;
}
else if (action_id == EE_ACTION_IfThenElse)
{
    [ExtendedEventCondition] condition;
    [ExtendedEventActionList] then_actions;
    [ExtendedEventActionList] else_actions;
}
else if (action_id == EE_ACTION_IfThenElseIf)
{
    [ExtendedEventIfThenElseIfPair] first_pair;
    [ExtendedEventIfThenElseIfTail] tail;
}
```

### ExtendedEventActionList

```
[uint32] action_list_marker;
[uint8] action_list_unknown;
[uint32] action_count;
[ExtendedEventAction] actions[action_count];
```

### ExtendedEventPlainActionList

```
[uint32] action_count;
[ExtendedEventAction] actions[action_count];
```

### ExtendedEventIfThenElseIfPair

```
[ExtendedEventCondition] condition;
[ExtendedEventActionList] then_actions;
```

### ExtendedEventIfThenElseIfTail

```
[uint8] chain_marker;
[uint32] has_next_pair;
if (has_next_pair != 0)
{
    [ExtendedEventIfThenElseIfPair] next_pair;
    [ExtendedEventIfThenElseIfTail] tail;
}
else
    [ExtendedEventPlainActionList] else_actions;
```

### ExtendedEventInfo

```
[uint32] id;

[uint8] unknown_bytes[5];

[uint32] action_count;

[ExtendedEventAction] actions[action_count];

[uint32] name_length;
[string] name[name_length];
```

### ExtendedEventSet

```
[uint32] event_count;

[ExtendedEventInfo] events[event_count];
```

### ExtendedEventCounters

```
[uint32] hero_id_counter;
[uint32] player_id_counter;
[uint32] town_id_counter;
[uint32] quest_id_counter;
[uint32] variable_id_counter;
```

### ExtendedEventIdList

```
[uint32] count;
[uint32] ids[count];
```

## 8. Settings after extended events

### Settings2

```
[uint32] artifact_count;
let artifact_bytes = (artifact_count + 7) / 8;
[uint8] artifacts[artifact_bytes];
[uint8] spells[9];
[uint8] skills[4];
```

## 9. Rumors

### Rumors

```
[uint32] rumor_count;
[Rumor] rumors[rumor_count];
```

### Rumor

```
[uint32] name_length;
[string] name[name_length];
[uint32] text_length;
[string] text[text_length];
```

## 10. Hero templates

Both hero arrays use the same slot order.

### HeroTemplates

```
[uint32] hero_count;
[HeroTemplate] heroes[hero_count];

[HeroTemplateHotA170] hota_170[hero_count];
```

### HeroTemplateHotA170

```
[uint8] add_skills;
[uint8] cannot_gain_xp;
[uint32] level;
```

### ArtifactRef

```
[uint16] id;
[uint16] spell_or_extra;
```

### SecondarySkill

```
[uint8] id;
[uint8] level;
```

### CreatureStack

```
[uint16] id;
[uint16] amount;
```

### CreatureStacks(amount)

```
[CreatureStack] stack[amount];
```

### HeroArtifactSlots

```
[ArtifactRef] head;
[ArtifactRef] shoulders;
[ArtifactRef] neck;
[ArtifactRef] right_hand;
[ArtifactRef] left_hand;
[ArtifactRef] torso;
[ArtifactRef] right_ring;
[ArtifactRef] left_ring;
[ArtifactRef] feet;
[ArtifactRef] misc_1;
[ArtifactRef] misc_2;
[ArtifactRef] misc_3;
[ArtifactRef] misc_4;
[ArtifactRef] war_machine_1;
[ArtifactRef] war_machine_2;
[ArtifactRef] war_machine_3;
[ArtifactRef] war_machine_4;
[ArtifactRef] spellbook;
[ArtifactRef] misc_5;
```

### HeroTemplate

```
[uint8] is_present;
if (is_present)
{
    [uint8] has_custom_experience;
    if (has_custom_experience)
        [uint32] experience;

    [uint8] has_custom_secondary_skills;
    if (has_custom_secondary_skills)
    {
        [uint32] secondary_skill_count;
        [SecondarySkill] secondary_skills[secondary_skill_count];
    }

    [uint8] has_artifacts;
    if (has_artifacts)
    {
        [HeroArtifactSlots] equipped;
        [uint16] backpack_count;
        [ArtifactRef] backpack[backpack_count];
    }

    [uint8] has_biography;
    if (has_biography)
        [uint32] biography_length;
        [string] biography[biography_length];

    [uint8] gender;

    [uint8] has_spells;
    if (has_spells)
        [uint8] spells[9];

    [uint8] has_primary_skills;
    if (has_primary_skills)
    {
        [uint8] attack;
        [uint8] defense;
        [uint8] spell_power;
        [uint8] knowledge;
    }
}
```

## 11. Terrain

Pass the header's map size and underground flag. The surface comes first. Within a layer, x varies fastest. The zero-based tile index is `(z * map_size + y) * map_size + x`. Each tile is seven bytes.

Terrain, river, and road IDs are listed in [Numeric identifiers](#numeric-identifiers).

### TerrainData(map_size, has_underground)

```
let tile_count = map_size * map_size;
if (has_underground)
    tile_count *= 2;

[TerrainTile] tiles[tile_count];
```

### TerrainTile

```
[uint8] terrain_type;
[uint8] terrain_sprite;
[uint8] river_type;
[uint8] river_sprite;
[uint8] road_type;
[uint8] road_sprite;
[uint8] mirroring;
```

## 12. Object definitions

Definitions are shared by placed objects. A set yellow bit marks a visitable tile. When the yellow bit is clear, an unset red bit marks a blocked tile.

### ObjectDefs

```
[uint32] object_def_count;
[ObjectDef] defs[object_def_count];
```

### ObjectDef

```
[uint32] sprite_length;
[string] sprite[sprite_length];
[uint8] red_squares[6];
[uint8] yellow_squares[6];
[uint8] placeable_terrain[2];
[uint8] editor_section[2];
[uint32] id;
[uint32] sub_id;
[uint8] editor_group;
[uint8] below_ground;
[uint8] null_bytes[16];
```

## 13. Placed objects

Resolve each object's zero-based `def_id` against the preceding object definitions. The resulting object type and subtype select the payload. There is no per-object payload length. A defined type or subtype with no payload branch has zero payload bytes.

### ObjectData

```
[uint32] object_count;
[ObjectDataEntry] objects[object_count];
```

### CommonPayload

```
[uint32] message_length;
[string] message[message_length];

[uint8] has_guards;
if (has_guards)
    [CreatureStack] guards[7];

[uint8] common_garbage_bytes[4];
```

### OptionalCommon

```
[uint8] has_common;
if (has_common)
    [CommonPayload] common;
```

### ObjectContents

```
[uint32] experience;
[uint32] spell_points;
[int8] morale;
[int8] luck;
[int32] resources[7];
[int8] primary_skills[4];

[uint8] secondary_skill_count;
[SecondarySkill] secondary_skills[secondary_skill_count];

[uint8] artifact_count;
[ArtifactRef] artifacts[artifact_count];

[uint8] spell_count;
[uint8] spells[spell_count];

[uint8] creature_count;
[CreatureStack] creatures[creature_count];

[uint8] garbage_bytes[8];
```

### MovementPayload

```
[uint32] movement_mode;
[uint32] movement_points;
```

### ObjectArtifactPayload

```
[OptionalCommon] common;
[uint32] pickup_mode;
[uint8] pickup_conditions;
```

### PandorasBoxPayload

```
[OptionalCommon] common;
[ObjectContents] contents;
[uint8] unknown_movement_prefix;
[MovementPayload] movement;
[uint8] difficulty[4];
[uint8] has_hota_event;
if (has_hota_event)
{
    [uint32] hero_event_id;
    [uint8] unknown_byte;
}
```

### BlackMarketPayload

```
[ArtifactRef] artifacts[7];
```

### CampfirePayload

```
[uint32] mode;
[uint8] extra_bytes[4];
[CampfireResource] resources[2];
```

### CampfireResource

```
[uint32] amount;
[uint8] resource_id;
```

### BankPayload

```
[uint32] difficulty;
[uint8] upgraded_stack;
[uint32] number_of_artifacts;
[uint32] artifacts[number_of_artifacts];
```

### CorpsePayload

```
[uint32] contents;
[uint32] artifact;
```

### EventObjectPayload

```
[OptionalCommon] common;
[ObjectContents] contents;
[uint8] allowed_players;
[uint8] allow_ai;
[uint8] cancel_event;
[uint8] garbage_bytes[4];
[uint8] allow_human;
[MovementPayload] movement;
[uint8] difficulty[4];
[uint8] has_hota_event;
if (has_hota_event)
{
    [uint32] hero_event_id;
    [uint8] unknown_byte;
}
```

### FourAndFourPayload

```
[uint32] contents;
[uint8] bytes[4];
```

### LeanToPayload

```
[uint32] contents;
[uint8] trash_bytes[4];
[uint32] amount;
[uint8] resource;
[uint8] mystery_bytes[5];
```

### GarrisonPayload

```
[uint32] owner;
[CreatureStack] guards[7];
[uint8] troops_removable;
[uint8] skipped_bytes[8];
```

### MapHeroPayload

```
[uint8] start_bytes[4];
[uint8] owner;
[uint8] hero_id;

[uint8] has_custom_name;
if (has_custom_name)
    [uint32] custom_name_length;
    [string] custom_name[custom_name_length];

[uint8] has_experience;
if (has_experience)
    [uint32] experience;

[uint8] has_portrait;
if (has_portrait)
    [uint8] portrait_id;

[uint8] has_secondary_skills;
if (has_secondary_skills)
{
    [uint32] secondary_skill_count;
    [SecondarySkill] secondary_skills[secondary_skill_count];
}

[uint8] has_army;
if (has_army)
    [CreatureStack] creatures[7];

[uint8] formation;

[uint8] has_artifacts;
if (has_artifacts)
{
    [HeroArtifactSlots] equipped;
    [uint16] backpack_count;
    [ArtifactRef] backpack[backpack_count];
}

[uint8] patrol;

[uint8] has_biography;
if (has_biography)
    [uint32] biography_length;
    [string] biography[biography_length];

[uint8] gender;

[uint8] has_spells;
if (has_spells)
    [uint8] spells[9];

[uint8] has_primary_skills;
if (has_primary_skills)
{
    [uint8] attack;
    [uint8] defense;
    [uint8] spell_power;
    [uint8] knowledge;
}

[uint8] end_bytes[16];
[uint8] add_skills;
[uint8] cannot_gain_xp;
[uint32] level;
```

### HeroPlaceholderPayload

```
[uint8] owner;
[uint8] hero_id;
if (hero_id == 255)
    [uint8] power_rating;
```

### MonsterPayload

```
[uint8] start_bytes[4];
[uint16] quantity;
[uint8] disposition;

[uint8] has_message;
if (has_message)
{
    [uint32] message_length;
    [string] message[message_length];
    [int32] resources[7];
    [uint16] artifact;
}

[uint8] monster_never_flees;
[uint8] quantity_does_not_grow;
[uint8] middle_bytes[2];
[uint32] precise_disposition;
[uint8] join_only_for_money;
[uint32] joining_monster_percent;
[uint32] upgraded_stack;
[uint32] stack_count;
[uint8] is_value;
[uint32] ai_value;
```

### TownBuildingBits

```
[uint8] bits[6];
```

### TownPayload

```
[uint8] start_bytes[4];
[uint8] owner;

[uint8] has_name;
if (has_name)
    [uint32] name_length;
    [string] name[name_length];

[uint8] garrison_customized;
if (garrison_customized)
    [CreatureStack] garrison_guards[7];

[uint8] garrison_formation;

[uint8] has_custom_buildings;
if (has_custom_buildings)
{
    [TownBuildingBits] buildings_built;
    [TownBuildingBits] buildings_disabled;
}
else
    [uint8] has_fort;

[uint8] spells_must_appear[9];
[uint8] spells_cant_appear[9];
[uint8] spell_research;

[uint32] special_building_count;
[uint8] special_buildings[special_building_count];

[EventList] town_events(1);
[uint8] alignment;
[uint8] skipped_bytes[3];
```

### ResourcePayload

```
[OptionalCommon] common;
[uint32] amount;
[uint8] garbage_bytes[4];
```

### ScholarPayload

```
[uint8] reward_type;
[uint8] reward_value_or_zero;
[uint8] skipped_bytes[6];
```

### Quest

```
[uint8] type;

if (type == QUEST_AchieveExperienceLevel)
    [uint32] experience_level;
else if (type == QUEST_AchievePrimarySkillLevel)
    [uint8] primary_skills[4];
else if (type == QUEST_DefeatSpecificHero ||
         type == QUEST_DefeatSpecificMonster)
    [uint8] object_identifier[4];
else if (type == QUEST_ReturnWithArtifacts)
{
    [uint8] artifact_count;
    [ArtifactRef] artifacts[artifact_count];
}
else if (type == QUEST_ReturnWithCreatures)
{
    [uint8] creature_count;
    [CreatureStack] creatures[creature_count];
}
else if (type == QUEST_ReturnWithResources)
    [int32] resources[7];
else if (type == QUEST_BeSpecificHero)
    [uint8] hero_id;
else if (type == QUEST_BelongToSpecificPlayer)
    [uint8] player_id;
else if (type == QUEST_HotAQuest)
{
    [uint32] hota_type;

    if (hota_type == HOTA_Q_BelongToSpecificClass)
    {
        [uint32] hota_extra;
        [uint8] allowed_classes[3];
    }
    else if (hota_type == HOTA_Q_ReturnNotBeforeDate ||
             hota_type == HOTA_Q_PlayOnDifficulty)
        [uint32] value;
    else if (hota_type == HOTA_Q_ExtendedEvent)
    {
        [uint32] hota_quest_event_id;
        [uint8] unknown_byte;
    }
}

if (type != QUEST_None)
{
    [uint32] deadline;
    [uint32] proposal_message_length;
    [string] proposal_message[proposal_message_length];
    [uint32] progress_message_length;
    [string] progress_message[progress_message_length];
    [uint32] completion_message_length;
    [string] completion_message[completion_message_length];
    if (general_specs.hota_map_version >= 10)
        [uint32] timeout_message_length;
        [string] timeout_message[timeout_message_length];
}
```

### Reward

```
[uint8] type;

if (type == REWARD_Experience || type == REWARD_SpellPoints)
    [uint32] value;
else if (type == REWARD_Morale || type == REWARD_Luck || type == REWARD_Spell)
    [uint8] value;
else if (type == REWARD_Resource)
{
    [uint8] resource;
    [uint32] amount;
}
else if (type == REWARD_PrimarySkill || type == REWARD_SecondarySkill)
{
    [uint8] id;
    [uint8] level;
}
else if (type == REWARD_Artifact)
    [ArtifactRef] artifact;
else if (type == REWARD_Creatures)
    [CreatureStack] creature;
```

### SeersHutQuestReward

```
[Quest] quest;
[Reward] reward;
```

### SeersHutPayload

```
[uint32] one_time_quest_count;
[SeersHutQuestReward] one_time_quests[one_time_quest_count];

[uint32] repeatable_quest_count;
[SeersHutQuestReward] repeatable_quests[repeatable_quest_count];

if (general_specs.hota_map_version >= 10)
{
    [uint8] has_custom_name;
    if (has_custom_name)
        [uint32] custom_name_length;
        [string] custom_name[custom_name_length];
}

[uint8] skipped_bytes[2];
```

### TreeOrSimpleArtifactPayload

```
[uint32] contents;
[uint32] artifact_or_extra;
```

### UniversityPayload

```
[uint32] mode;
[uint8] skills[4];
```

### WagonLikePayload

```
[uint32] contents;
[uint32] artifact;
[uint32] amount;
[uint8] resource;
[uint8] mystery_bytes[5];
```

### DwellingPayload

```
[uint32] owner;
[uint32] same_as_town;
if (same_as_town == 0)
    [uint8] alignment[2];
[uint8] minimum_level;
[uint8] maximum_level;
```

### LeveledDwellingPayload

```
[uint32] owner;
[uint32] same_as_town;
if (same_as_town == 0)
    [uint8] alignment[2];
```

### FactionDwellingPayload

```
[uint32] owner;
[uint8] minimum_level;
[uint8] maximum_level;
```

### AncientLampPayload

```
[uint32] contents;
[uint8] trash_bytes[4];
[uint32] amount;
[uint8] mystery_bytes[6];
```

### SeaBarrelPayload

```
[uint32] contents;
[uint8] trash_bytes[4];
[uint32] amount;
[uint8] resource;
[uint8] mystery_bytes[5];
```

### VialOfManaPayload

```
[uint32] contents;
[uint8] trash_bytes[4];
```

### AbandonedMinePayload

```
[uint8] resources;
[uint8] mid_bytes[3];
[uint8] is_custom;
[uint32] creature;
[uint32] min_val;
[uint32] max_val;
```

### TrapperLodgePayload

```
[uint32] reward_type;
[uint32] gold_amount;
[uint32] creature_amount;
[uint32] creature_id;
```

### ObjectDataEntry

```
[Coords] coords;
[uint32] def_id;
[uint8] skipped_bytes[5];

let object_id = defs[def_id].id;
let sub_id = defs[def_id].sub_id;

if (object_id == OBJ_PandorasBox)
    [PandorasBoxPayload] pandoras_box;
else if (object_id == OBJ_BlackMarket)
    [BlackMarketPayload] black_market;
else if (object_id == OBJ_Campfire)
    [CampfirePayload] campfire;
else if (object_id == OBJ_Corpse)
    [CorpsePayload] corpse;
else if (object_id == OBJ_EventObject)
    [EventObjectPayload] event_object;
else if (object_id == OBJ_Flotsam)
    [FourAndFourPayload] flotsam;
else if (object_id == OBJ_LeanTo)
    [LeanToPayload] lean_to;
else if (object_id == OBJ_Pyramid)
    [FourAndFourPayload] pyramid;
else if (object_id == OBJ_Scholar)
    [ScholarPayload] scholar;
else if (object_id == OBJ_SeaChest)
    [TreeOrSimpleArtifactPayload] sea_chest;
else if (object_id == OBJ_SeersHut)
    [SeersHutPayload] seers_hut;
else if (object_id == OBJ_ShipwreckSurvivor)
    [TreeOrSimpleArtifactPayload] shipwreck_survivor;
else if (object_id == OBJ_TreasureChest)
    [TreeOrSimpleArtifactPayload] treasure_chest;
else if (object_id == OBJ_TreeOfKnowledge)
    [TreeOrSimpleArtifactPayload] tree_of_knowledge;
else if (object_id == OBJ_University)
    [UniversityPayload] university;
else if (object_id == OBJ_Wagon)
    [WagonLikePayload] wagon;
else if (object_id == OBJ_WarriorsTomb)
    [TreeOrSimpleArtifactPayload] warriors_tomb;
else if (object_id == OBJ_RandomDwelling)
    [DwellingPayload] random_dwelling;
else if (object_id == OBJ_RandomDwellingLeveled)
    [LeveledDwellingPayload] random_dwelling_leveled;
else if (object_id == OBJ_RandomDwellingFaction)
    [FactionDwellingPayload] random_dwelling_faction;
else if (object_id == OBJ_QuestGuard)
    [Quest] quest_guard_quest;
else if (object_id == OBJ_Grail)
    [uint32] grail_radius;
else if (object_id == OBJ_WitchHut)
    [uint8] witch_hut_skills[4];
else if (object_id == OBJ_HotAPickup && sub_id == 0)
    [AncientLampPayload] ancient_lamp;
else if (object_id == OBJ_HotAPickup && sub_id == 1)
    [SeaBarrelPayload] sea_barrel;
else if (object_id == OBJ_HotAPickup && sub_id == 2)
    [FourAndFourPayload] jetsam;
else if (object_id == OBJ_HotAPickup && sub_id == 3)
    [VialOfManaPayload] vial_of_mana;
else if (object_id == OBJ_AbandonedMine)
    [AbandonedMinePayload] abandoned_mine;
else if (object_id == OBJ_Mine && sub_id == 7)
    [AbandonedMinePayload] abandoned_mine_from_mine;
else if (object_id == OBJ_Mine)
    [uint32] mine_owner;
else if (object_id == OBJ_HotAVisitable1 && sub_id == 12)
    [TrapperLodgePayload] trapper_lodge;
else if (object_id == OBJ_HotAVisitable2 && sub_id == 0)
    [UniversityPayload] seafaring_academy;
else if (object_id == OBJ_BorderGate && sub_id == 1000)
    [Quest] hota_quest_gate_quest;
else if (object_id == OBJ_BorderGate && sub_id == 1001)
    [WagonLikePayload] grave;
else if (object_id == OBJ_Town || object_id == OBJ_RandomTown)
    [TownPayload] town;
else if (object_id == OBJ_Resource || object_id == OBJ_RandomResource)
    [ResourcePayload] resource;
else if (object_id == OBJ_Hero || object_id == OBJ_Prison || object_id == OBJ_RandomHero)
    [MapHeroPayload] hero;
else if (object_id in {
    OBJ_Monster, OBJ_RandomMonster, OBJ_RandomMonster1,
    OBJ_RandomMonster2, OBJ_RandomMonster3, OBJ_RandomMonster4,
    OBJ_RandomMonster5, OBJ_RandomMonster6, OBJ_RandomMonster7
})
    [MonsterPayload] monster;
else if (object_id in {
    OBJ_Artifact, OBJ_RandomArtifact, OBJ_RandomTreasureArtifact,
    OBJ_RandomMinorArtifact, OBJ_RandomMajorArtifact, OBJ_RandomRelic
})
    [ObjectArtifactPayload] artifact;
else if (object_id == OBJ_OceanBottle || object_id == OBJ_Sign)
{
    [uint32] message_length;
    [string] message[message_length];
    [uint8] garbage_bytes[4];
}
else if (object_id == OBJ_CreatureDwellingNormal ||
         object_id == OBJ_Lighthouse ||
         object_id == OBJ_CreatureDwellingMulti ||
         object_id == OBJ_Shipyard)
    [uint32] owner;
else if (object_id == OBJ_Garrison || object_id == OBJ_GarrisonVertical)
    [GarrisonPayload] garrison;
else if (object_id == OBJ_Shrine1And4 ||
         object_id == OBJ_ShrineOfMagicGesture ||
         object_id == OBJ_ShrineOfMagicThought)
    [uint32] shrine_spell;
else if (object_id == OBJ_SpellScroll)
{
    [OptionalCommon] common;
    [uint32] spell;
}
else if (object_id == OBJ_HeroPlaceholder)
    [HeroPlaceholderPayload] hero_placeholder;
else if (object_id in {
    OBJ_CreatureBank, OBJ_DerelictShip, OBJ_DragonUtopia,
    OBJ_Crypt, OBJ_Shipwreck
})
    [BankPayload] bank;
```

## 14. Global events

Pass `is_town = 0` here and `is_town = 1` for a town's embedded event list.

### EventList(is_town)

```
[uint32] event_count;
for (i = 0; i < event_count; i++)
    [MapEvent] event(is_town);
```

### MapEvent(is_town)

```
[uint32] name_length;
[string] name[name_length];
[uint32] message_length;
[string] message[message_length];
[int32] resources[7];
[uint8] apply_to;
[uint8] apply_human;
[uint8] apply_ai;
[uint16] first_occurrence;
[uint16] subsequent_occurrences;
[uint8] trash_bytes[16];
[uint32] allowed_difficulties;

[uint8] event_type;
if (is_town)
{
    if (event_type == 0)
        [uint32] hota_level_7b;
    else if (event_type == 1)
    {
        [uint32] hota_town_event_id;
        [uint8] unknown_bytes[5];
    }

    [uint32] hota_amount;
    [uint8] hota_special[6];
    [uint8] apply_neutral_towns;
    [uint8] buildings[6];
    [uint16] creatures[7];
    [uint8] end_trash[4];
}
else
{
    if (event_type == 1)
        [uint8] ext_bytes[5];
}
```

## 15. Trailer

Any bytes after the 124-byte trailer are opaque.

```
[uint8] null_bytes[124];
```

## Numeric identifiers

Combine each indicated prefix with the table entry to obtain its layout symbol. Field layouts specify the stored integer types.

### Variable modes

Prefix `VAR_`.

```
InitialValue       = 0
ImportFromPrevious = 1
```

### Map formats

```
RoE  = 14
AB   = 21
SoD  = 28
CHR  = 29
HotA = 32
WoG  = 51
```

### Map sizes

```
Small      = 36
Medium     = 72
Large      = 108
ExtraLarge = 144
Huge       = 180
ExtraHuge  = 216
Giant      = 252
```

### Difficulty levels

```
Easy       = 0
Normal     = 1
Hard       = 2
Expert     = 3
Impossible = 4
```

### Player color IDs

```
Red    = 0
Blue   = 1
Tan    = 2
Green  = 3
Orange = 4
Purple = 5
Teal   = 6
Pink   = 7
```

### Town types

```
Castle     = 0
Rampart    = 1
Tower      = 2
Inferno    = 3
Necropolis = 4
Dungeon    = 5
Stronghold = 6
Fortress   = 7
Conflux    = 8
Cove       = 9
Factory    = 10
Bulwark    = 11
Random     = 255
```

### Victory conditions

Prefix `VC_`.

```
AcquireArtifact     = 0
AccumulateCreatures = 1
AccumulateResources = 2
UpgradeTown         = 3
BuildTheGrail       = 4
DefeatHero          = 5
CaptureTown         = 6
DefeatMonster       = 7
FlagDwellings       = 8
FlagMines           = 9
TransportArtifact   = 10
EliminateMonsters   = 11
Survive             = 12
None                = 255
```

### Loss conditions

Prefix `LC_`.

```
LoseTown    = 0
LoseHero    = 1
TimeExpires = 2
None        = 255
```

### Object IDs

Prefix `OBJ_`.

```
AltarOfSacrifice       = 2
AnchorPoint            = 3
Arena                  = 4
Artifact               = 5
PandorasBox            = 6
BlackMarket            = 7
Boat                   = 8
BorderGuard            = 9
KeymastersTent         = 10
Buoy                   = 11
Campfire               = 12
Cartographer           = 13
SwanPond               = 14
CoverOfDarkness        = 15
CreatureBank           = 16
CreatureDwellingNormal = 17
Unknown18              = 18
Unknown19              = 19
CreatureDwellingMulti  = 20
CursedGroundRoE        = 21
Corpse                 = 22
MarlettoTower          = 23
DerelictShip           = 24
DragonUtopia           = 25
EventObject            = 26
EyeOfTheMagi           = 27
FaerieRing             = 28
Flotsam                = 29
FountainOfFortune      = 30
FountainOfYouth        = 31
GardenOfRevelation     = 32
Garrison               = 33
Hero                   = 34
HillFort               = 35
Grail                  = 36
HutOfTheMagi           = 37
IdolOfFortune          = 38
LeanTo                 = 39
LibraryOfEnlightenment = 41
Lighthouse             = 42
OneWayMonolithEntrance = 43
OneWayMonolithExit     = 44
TwoWayMonolith         = 45
MagicPlainsRoE         = 46
SchoolOfMagic          = 47
MagicSpring            = 48
MagicWell              = 49
MarketOfTime           = 50
MercenaryCamp          = 51
Mermaids               = 52
Mine                   = 53
Monster                = 54
MysticalGarden         = 55
Oasis                  = 56
Obelisk                = 57
RedwoodObservatory     = 58
OceanBottle            = 59
PillarOfFire           = 60
StarAxis               = 61
Prison                 = 62
Pyramid                = 63
RallyFlag              = 64
RandomArtifact         = 65
RandomTreasureArtifact = 66
RandomMinorArtifact    = 67
RandomMajorArtifact    = 68
RandomRelic            = 69
RandomHero             = 70
RandomMonster          = 71
RandomMonster1         = 72
RandomMonster2         = 73
RandomMonster3         = 74
RandomMonster4         = 75
RandomResource         = 76
RandomTown             = 77
RefugeeCamp            = 78
Resource               = 79
Sanctuary              = 80
Scholar                = 81
SeaChest               = 82
SeersHut               = 83
Crypt                  = 84
Shipwreck              = 85
ShipwreckSurvivor      = 86
Shipyard               = 87
Shrine1And4            = 88
ShrineOfMagicGesture   = 89
ShrineOfMagicThought   = 90
Sign                   = 91
Sirens                 = 92
SpellScroll            = 93
Stables                = 94
Tavern                 = 95
Temple                 = 96
DenOfThieves           = 97
Town                   = 98
TradingPost            = 99
LearningStone          = 100
TreasureChest          = 101
TreeOfKnowledge        = 102
SubterraneanGate       = 103
University             = 104
Wagon                  = 105
WarMachineFactory      = 106
SchoolOfWar            = 107
WarriorsTomb           = 108
WaterWheel             = 109
WateringHole           = 110
Whirlpool              = 111
Windmill               = 112
WitchHut               = 113
Brush                  = 114
Bush                   = 115
Cactus                 = 116
Canyon                 = 117
Crater                 = 118
DeadVegetation         = 119
Flowers                = 120
FrozenLake             = 121
Hedge                  = 122
Hill                   = 123
Hole                   = 124
Kelp                   = 125
Lake                   = 126
LavaFlow               = 127
LavaLake               = 128
Mushrooms              = 129
Log                    = 130
Mandrake               = 131
Moss                   = 132
Mound                  = 133
Mountain               = 134
OakTrees               = 135
Outcropping            = 136
PineTrees              = 137
Plant                  = 138
HotADecor1             = 139
HotADecor2             = 140
HotAMagicalTerrain     = 141
HotAWarehouse          = 142
RiverDelta             = 143
HotAVisitable1         = 144
HotAPickup             = 145
HotAVisitable2         = 146
Rock                   = 147
SandDune               = 148
SandPit                = 149
Shrub                  = 150
Skull                  = 151
Stalagmite             = 152
Stump                  = 153
TarPit                 = 154
Trees                  = 155
Vine                   = 156
VolcanicVent           = 157
Volcano                = 158
WillowTrees            = 159
YuccaTrees             = 160
Reef                   = 161
RandomMonster5         = 162
RandomMonster6         = 163
RandomMonster7         = 164
Brush2                 = 165
Bush2                  = 166
Cactus2                = 167
Canyon2                = 168
Crater2                = 169
DeadVegetation2        = 170
Flowers2               = 171
FrozenLake2            = 172
Hedge2                 = 173
Hill2                  = 174
Hole2                  = 175
Kelp2                  = 176
Lake2                  = 177
LavaFlow2              = 178
LavaLake2              = 179
Mushrooms2             = 180
Log2                   = 181
Mandrake2              = 182
Moss2                  = 183
Mound2                 = 184
Mountain2              = 185
OakTrees2              = 186
Outcropping2           = 187
PineTrees2             = 188
Plant2                 = 189
RiverDelta2            = 190
Rock2                  = 191
SandDune2              = 192
SandPit2               = 193
Shrub2                 = 194
Skull2                 = 195
Stalagmite2            = 196
Stump2                 = 197
TarPit2                = 198
Trees2                 = 199
Vine2                  = 200
VolcanicVent2          = 201
Volcano2               = 202
WillowTrees2           = 203
YuccaTrees2            = 204
Reef2                  = 205
DesertHills            = 206
DirtHills              = 207
GrassHills             = 208
RoughHills             = 209
SubterraneanRocks      = 210
SwampFoliage           = 211
BorderGate             = 212
FreelancersGuild       = 213
HeroPlaceholder        = 214
QuestGuard             = 215
RandomDwelling         = 216
RandomDwellingLeveled  = 217
RandomDwellingFaction  = 218
GarrisonVertical       = 219
AbandonedMine          = 220
TradingPostSnow        = 221
CloverField            = 222
CursedGround           = 223
EvilFog                = 224
FavorableWinds         = 225
FieryFields            = 226
HolyGround             = 227
LucidPools             = 228
MagicClouds            = 229
MagicPlains            = 230
Rocklands              = 231
```

### Quest types

Prefix `QUEST_`.

```
None                     = 0
AchieveExperienceLevel   = 1
AchievePrimarySkillLevel = 2
DefeatSpecificHero       = 3
DefeatSpecificMonster    = 4
ReturnWithArtifacts      = 5
ReturnWithCreatures      = 6
ReturnWithResources      = 7
BeSpecificHero           = 8
BelongToSpecificPlayer   = 9
HotAQuest                = 10
```

### HotA quest types

Prefix `HOTA_Q_`.

```
BelongToSpecificClass = 0
ReturnNotBeforeDate   = 1
PlayOnDifficulty      = 2
ExtendedEvent         = 3
```

### Reward types

Prefix `REWARD_`.

```
None           = 0
Experience     = 1
SpellPoints    = 2
Morale         = 3
Luck           = 4
Resource       = 5
PrimarySkill   = 6
SecondarySkill = 7
Artifact       = 8
Spell          = 9
Creatures      = 10
```

### Terrain IDs

```
Dirt         = 0
Sand         = 1
Grass        = 2
Snow         = 3
Swamp        = 4
Rough        = 5
Subterranean = 6
Lava         = 7
Water        = 8
Rock         = 9
Highlands    = 10
Wasteland    = 11
```

### River IDs

```
None  = 0
Clear = 1
Icy   = 2
Muddy = 3
Lava  = 4
```

### Road IDs

```
None        = 0
Dirt        = 1
Gravel      = 2
Cobblestone = 3
```

### Resource IDs

```
Wood    = 0
Mercury = 1
Ore     = 2
Sulfur  = 3
Crystal = 4
Gems    = 5
Gold    = 6
```

### Difficulty icons

```
Pawn   = 0
Knight = 1
Rook   = 2
Queen  = 3
King   = 4
```

### Primary skill IDs

```
Attack     = 0
Defense    = 1
SpellPower = 2
Knowledge  = 3
```

### Artifact IDs

Prefix `ARTIFACT_`.

```
SpellBook                   = 0
SpellScroll                 = 1
TheGrail                    = 2
Catapult                    = 3
Ballista                    = 4
AmmoCart                    = 5
FirstAidTent                = 6
CentaursAxe                 = 7
BlackshardOfTheDeadKnight   = 8
GreaterGnollsFlail          = 9
OgresClubOfHavoc            = 10
SwordOfHellfire             = 11
TitansGladius               = 12
ShieldOfTheDwarvenLords     = 13
ShieldOfTheYawningDead      = 14
BucklerOfTheGnollKing       = 15
TargOfTheRampagingOgre      = 16
ShieldOfTheDamned           = 17
SentinelsShield             = 18
HelmOfTheAlabasterUnicorn   = 19
SkullHelmet                 = 20
HelmOfChaos                 = 21
CrownOfTheSupremeMagi       = 22
HellstormHelmet             = 23
ThunderHelmet               = 24
BreastplateOfPetrifiedWood  = 25
RibCage                     = 26
ScalesOfTheGreaterBasilisk  = 27
TunicOfTheCyclopsKing       = 28
BreastplateOfBrimstone      = 29
TitansCuirass               = 30
ArmorOfWonder               = 31
SandalsOfTheSaint           = 32
CelestialNecklaceOfBliss    = 33
LionsShieldOfCourage        = 34
SwordOfJudgement            = 35
HelmOfHeavenlyEnlightenment = 36
QuietEyeOfTheDragon         = 37
RedDragonFlameTongue        = 38
DragonScaleShield           = 39
DragonScaleArmor            = 40
DragonboneGreaves           = 41
DragonWingTabard            = 42
NecklaceOfDragonteeth       = 43
CrownOfDragontooth          = 44
StillEyeOfTheDragon         = 45
CloverOfFortune             = 46
CardsOfProphecy             = 47
LadybirdOfLuck              = 48
BadgeOfCourage              = 49
CrestOfValor                = 50
GlyphOfGallantry            = 51
Speculum                    = 52
Spyglass                    = 53
AmuletOfTheUndertaker       = 54
VampiresCowl                = 55
DeadMansBoots               = 56
GarnitureOfInterference     = 57
SurcoatOfCounterpoise       = 58
BootsOfPolarity             = 59
BowOfElvenCherrywood        = 60
BowstringOfTheUnicornsMane  = 61
AngelFeatherArrows          = 62
BirdOfPerception            = 63
StoicWatchman               = 64
EmblemOfCognizance          = 65
StatesmansMedal             = 66
DiplomatsRing               = 67
AmbassadorsSash             = 68
RingOfTheWayfarer           = 69
EquestriansGloves           = 70
NecklaceOfOceanGuidance     = 71
AngelWings                  = 72
CharmOfMana                 = 73
TalismanOfMana              = 74
MysticOrbOfMana             = 75
CollarOfConjuring           = 76
RingOfConjuring             = 77
CapeOfConjuring             = 78
OrbOfTheFirmament           = 79
OrbOfSilt                   = 80
OrbOfTempestuousFire        = 81
OrbOfDrivingRain            = 82
RecantersCloak              = 83
SpiritOfOppression          = 84
HourglassOfTheEvilHour      = 85
TomeOfFireMagic             = 86
TomeOfAirMagic              = 87
TomeOfWaterMagic            = 88
TomeOfEarthMagic            = 89
BootsOfLevitation           = 90
GoldenBow                   = 91
SphereOfPermanence          = 92
OrbOfVulnerability          = 93
RingOfVitality              = 94
RingOfLife                  = 95
VialOfLifeblood             = 96
NecklaceOfSwiftness         = 97
BootsOfSpeed                = 98
CapeOfVelocity              = 99
PendantOfDispassion         = 100
PendantOfSecondSight        = 101
PendantOfHoliness           = 102
PendantOfLife               = 103
PendantOfDeath              = 104
PendantOfFreeWill           = 105
PendantOfNegativity         = 106
PendantOfTotalRecall        = 107
PendantOfCourage            = 108
EverflowingCrystalCloak     = 109
RingOfInfiniteGems          = 110
EverpouringVialOfMercury    = 111
InexhaustibleCartOfOre      = 112
EversmokingRingOfSulfur     = 113
InexhaustibleCartOfLumber   = 114
EndlessSackOfGold           = 115
EndlessBagOfGold            = 116
EndlessPurseOfGold          = 117
LegsOfLegion                = 118
LoinsOfLegion               = 119
TorsoOfLegion               = 120
ArmsOfLegion                = 121
HeadOfLegion                = 122
SeaCaptainsHat              = 123
SpellbindersHat             = 124
ShacklesOfWar               = 125
OrbOfInhibition             = 126
VialOfDragonblood           = 127
ArmageddonsBlade            = 128
AngelicAlliance             = 129
CloakOfTheUndeadKing        = 130
ElixirOfLife                = 131
ArmorOfTheDamned            = 132
StatueOfLegion              = 133
PowerOfTheDragonFather      = 134
TitansThunder               = 135
AdmiralsHat                 = 136
BowOfTheSharpshooter        = 137
WizardsWell                 = 138
RingOfTheMagi               = 139
Cornucopia                  = 140
DiplomatsCloak              = 141
PendantOfReflection         = 142
IronfistOfTheOgre           = 143
Cannon                      = 146
TridentOfDominion           = 147
ShieldOfNavalGlory          = 148
RoyalArmorOfNix             = 149
CrownOfTheFiveSeas          = 150
WayfarersBoots              = 151
RunesOfImminency            = 152
DemonsHorseshoe             = 153
ShamansPuppet               = 154
HideousMask                 = 155
RingOfSuppression           = 156
PendantOfDownfall           = 157
RingOfOblivion              = 158
CapeOfSilence               = 159
GoldenGoose                 = 160
HornOfTheAbyss              = 161
CharmOfEclipse              = 162
SealOfSunset                = 163
PlateOfDyingLight           = 164
Sleepkeeper                 = 165
```

### Spell IDs

```
SummonBoat           = 0
ScuttleBoat          = 1
Visions              = 2
ViewEarth            = 3
Disguise             = 4
ViewAir              = 5
Fly                  = 6
WaterWalk            = 7
DimensionDoor        = 8
TownPortal           = 9
QuickSand            = 10
LandMine             = 11
ForceField           = 12
FireWall             = 13
Earthquake           = 14
MagicArrow           = 15
IceBolt              = 16
LightningBolt        = 17
Implosion            = 18
ChainLightning       = 19
FrostRing            = 20
Fireball             = 21
Inferno              = 22
MeteorShower         = 23
DeathRipple          = 24
DestroyUndead        = 25
Armageddon           = 26
Shield               = 27
AirShield            = 28
FireShield           = 29
ProtectionFromAir    = 30
ProtectionFromFire   = 31
ProtectionFromWater  = 32
ProtectionFromEarth  = 33
AntiMagic            = 34
Dispel               = 35
MagicMirror          = 36
Cure                 = 37
Resurrection         = 38
AnimateDead          = 39
Sacrifice            = 40
Bless                = 41
Curse                = 42
Bloodlust            = 43
Precision            = 44
Weakness             = 45
StoneSkin            = 46
DisruptingRay        = 47
Prayer               = 48
Mirth                = 49
Sorrow               = 50
Fortune              = 51
Misfortune           = 52
Haste                = 53
Slow                 = 54
Slayer               = 55
Frenzy               = 56
TitansLightningBolt  = 57
Counterstrike        = 58
Berserk              = 59
Hypnotize            = 60
Forgetfulness        = 61
Blind                = 62
Teleport             = 63
RemoveObstacle       = 64
Clone                = 65
SummonFireElemental  = 66
SummonEarthElemental = 67
SummonWaterElemental = 68
SummonAirElemental   = 69
```

### Hero class IDs

```
Knight       = 0
Cleric       = 1
Ranger       = 2
Druid        = 3
Alchemist    = 4
Wizard       = 5
Demoniac     = 6
Heretic      = 7
DeathKnight  = 8
Necromancer  = 9
Overlord     = 10
Warlock      = 11
Barbarian    = 12
BattleMage   = 13
Beastmaster  = 14
Witch        = 15
Planeswalker = 16
Elementalist = 17
Captain      = 18
Navigator    = 19
Mercenary    = 20
Artificer    = 21
Chieftain    = 22
Elder        = 23
```

### Secondary skill IDs

```
Pathfinding  = 0
Archery      = 1
Logistics    = 2
Scouting     = 3
Diplomacy    = 4
Navigation   = 5
Leadership   = 6
Wisdom       = 7
Mysticism    = 8
Luck         = 9
Ballistics   = 10
EagleEye     = 11
Necromancy   = 12
Estates      = 13
FireMagic    = 14
AirMagic     = 15
WaterMagic   = 16
EarthMagic   = 17
Scholar      = 18
Tactics      = 19
Artillery    = 20
Learning     = 21
Offense      = 22
Armorer      = 23
Intelligence = 24
Sorcery      = 25
Resistance   = 26
FirstAid     = 27
Interference = 28
Runes        = 29
```

### Secondary skill levels

```
Basic    = 1
Advanced = 2
Expert   = 3
```

### Hero IDs

```
Default     = 255
Orrin       = 0
Valeska     = 1
Edric       = 2
Sylvia      = 3
LordHaart   = 4
Sorsha      = 5
Christian   = 6
Tyris       = 7
Rion        = 8
Adela       = 9
Cuthbert    = 10
Adelaide    = 11
Ingham      = 12
Sanya       = 13
Loynis      = 14
Caitlin     = 15
Mephala     = 16
Ufretin     = 17
Jenova      = 18
Ryland      = 19
Thorgrim    = 20
Ivor        = 21
Clancy      = 22
Kyrre       = 23
Coronius    = 24
Uland       = 25
Elleshar    = 26
Gem         = 27
Malcom      = 28
Melodia     = 29
Alagar      = 30
Aeris       = 31
Piquedram   = 32
Thane       = 33
Josephine   = 34
Neela       = 35
Torosar     = 36
Fafner      = 37
Rissa       = 38
Iona        = 39
Astral      = 40
Halon       = 41
Serena      = 42
Daremyth    = 43
Theodorus   = 44
Solmyr      = 45
Cyra        = 46
Aine        = 47
Fiona       = 48
Rashka      = 49
Marius      = 50
Ignatius    = 51
Octavia     = 52
Calh        = 53
Pyre        = 54
Nymus       = 55
Ayden       = 56
Xyron       = 57
Axsis       = 58
Olema       = 59
Calid       = 60
Ash         = 61
Zydar       = 62
Xarfax      = 63
Straker     = 64
Vokial      = 65
Moandor     = 66
Charna      = 67
Tamika      = 68
Isra        = 69
Clavius     = 70
Galthran    = 71
Septienna   = 72
Aislinn     = 73
Sandro      = 74
Nimbus      = 75
Thant       = 76
Xsi         = 77
Vidomina    = 78
Nagash      = 79
Lorelei     = 80
Arlach      = 81
Dace        = 82
Ajit        = 83
Damacon     = 84
Gunnar      = 85
Synca       = 86
Shakti      = 87
Alamar      = 88
Jaegar      = 89
Malekith    = 90
Jeddite     = 91
Geon        = 92
Deemer      = 93
Sephinroth  = 94
Darkstorn   = 95
Yog         = 96
Gurnisson   = 97
Jabarkas    = 98
Shiva       = 99
Gretchin    = 100
Krellion    = 101
CragHack    = 102
Tyraxor     = 103
Gird        = 104
Vey         = 105
Dessa       = 106
Terek       = 107
Zubin       = 108
Gundula     = 109
Oris        = 110
Saurug      = 111
Bron        = 112
Drakon      = 113
Wystan      = 114
Tazar       = 115
Alkin       = 116
Korbac      = 117
Gerwulf     = 118
Broghild    = 119
Mirlanda    = 120
Rosic       = 121
Voy         = 122
Verdish     = 123
Merist      = 124
Styg        = 125
Andra       = 126
Tiva        = 127
Pasis       = 128
Thunar      = 129
Ignissa     = 130
Lacus       = 131
Monere      = 132
Erdamon     = 133
Fiur        = 134
Kalt        = 135
Luna        = 136
Brissa      = 137
Ciele       = 138
Labetha     = 139
Inteus      = 140
Aenain      = 141
Gelare      = 142
Grindan     = 143
SirMullich  = 144
Adrienne    = 145
Catherine   = 146
Dracon      = 147
Gelu        = 148
Kilgor      = 149
HaartLich   = 150
Mutare      = 151
Roland      = 152
MutareDrake = 153
Boragus     = 154
Xeron       = 155
Corkes      = 156
Jeremy      = 157
Illor       = 158
Derek       = 159
Leena       = 160
Anabel      = 161
Cassiopeia  = 162
Miriam      = 163
Casmetra    = 164
Eovacius    = 165
Spint       = 166
Andal       = 167
Manfred     = 168
Zilare      = 169
Astra       = 170
Dargem      = 171
Bidley      = 172
Tark        = 173
Elmore      = 174
Beatrice    = 175
Kinkeria    = 176
Ranloo      = 177
Giselle     = 178
Henrietta   = 179
Sam         = 180
Tancred     = 181
Melchior    = 182
Floribert   = 183
Wynona      = 184
Dury        = 185
Morton      = 186
Celestine   = 187
Todd        = 188
Agar        = 189
Bertram     = 190
Wrathmont   = 191
Ziph        = 192
Victoria    = 193
Eanswythe   = 194
Frederick   = 195
Tavin       = 196
Murdoch     = 197
Dhuin       = 198
Oidana      = 199
Neia        = 200
Eikthum     = 201
Creyle      = 202
Spadum      = 203
Kynr        = 204
Ergon       = 205
Kriv        = 206
Glacius     = 207
Sial        = 208
Dalton      = 209
Biarma      = 210
Akka        = 211
Vehr        = 212
Allora      = 213
Haugir      = 214
```

### Town building IDs

```
TownHall              = 0
CityHall              = 1
Capitol               = 2
Fort                  = 3
Citadel               = 4
Castle                = 5
Tavern                = 6
Blacksmith            = 7
Marketplace           = 8
ResourceSilo          = 9
ArtifactMerchants     = 10
MageGuildLevel1       = 11
MageGuildLevel2       = 12
MageGuildLevel3       = 13
MageGuildLevel4       = 14
MageGuildLevel5       = 15
Shipyard              = 16
Grail                 = 17
Special1              = 18
Special2              = 19
Special3              = 20
Special4              = 21
DwellingLevel1        = 22
DwellingLevel1Upgrade = 23
HordeLevel1           = 24
DwellingLevel2        = 25
DwellingLevel2Upgrade = 26
HordeLevel2           = 27
DwellingLevel3        = 28
DwellingLevel3Upgrade = 29
HordeLevel3           = 30
DwellingLevel4        = 31
DwellingLevel4Upgrade = 32
HordeLevel4           = 33
DwellingLevel5        = 34
DwellingLevel5Upgrade = 35
HordeLevel5           = 36
DwellingLevel6        = 37
DwellingLevel6Upgrade = 38
DwellingLevel7        = 39
DwellingLevel7Upgrade = 40
Building41            = 41
Building42            = 42
Building43            = 43
Building44            = 44
Building45            = 45
Building46            = 46
Building47            = 47
```

### Spell point operations

```
Give      = 0
Take      = 1
Nullify   = 2
Set       = 3
Replenish = 4
```

### Movement point operations

```
Give      = 0
Take      = 1
Nullify   = 2
Set       = 3
Replenish = 4
```

### Extended event action IDs

Prefix `EE_ACTION_`.

```
IfThenElseIf                   = 1
SetVariableBasedOnCondition    = 2
ModifyVariable                 = 3
ResourcesReward                = 4
RemoveCurrentObjectFinishQuest = 5
MessageWithReward              = 6
QuestAction                    = 7
CreaturesReward                = 8
ArtifactReward                 = 9
BuildTownStructure             = 10
SetQuestHintAndLogEntry        = 11
Question                       = 12
IfThenElse                     = 13
AddCreaturesToHire             = 14
SpellReward                    = 15
ExperienceReward               = 16
SpellPointsReward              = 17
MovementPointsReward           = 18
PrimarySkillsReward            = 19
SecondarySkillsReward          = 20
LuckReward                     = 21
MoraleReward                   = 22
Combat                         = 23
ExecuteEvent                   = 24
WarMachineReward               = 25
SpellbookReward                = 26
DisableEvent                   = 27
Cycle                          = 28
ShowMessage                    = 29
```

### Extended event types

```
Hero   = 0
Player = 1
Town   = 2
Quest  = 3
```

### Extended event variable operations

```
Add      = 0
Subtract = 1
Set      = 2
```

### Extended event expression types

Prefix `EE_EXPR_`.

```
IntegerValue           = 0
VariableValue          = 1
InvertSign             = 2
Add                    = 3
Subtract               = 4
Resource               = 5
Multiply               = 6
Divide                 = 7
Remainder              = 8
CreatureCount          = 9
CurrentDifficultyLevel = 10
DifficultyLevel        = 11
CurrentDate            = 12
CurrentHeroExperience  = 13
CurrentHeroLevel       = 14
PrimarySkill           = 15
RandomNumber           = 16
ArtifactCount          = 17
```

### Extended event expression operand modes

Prefix `EE_EXPR_OPERAND_`.

```
Integer    = 0
Expression = 1
```

### Extended event condition types

Prefix `EE_COND_`.

```
Boolean                 = 0
And                     = 1
Or                      = 2
LessThan                = 3
GreaterThan             = 4
Equal                   = 5
Not                     = 6
HeroHasArtifact         = 7
GreaterThanOrEqual      = 8
LessThanOrEqual         = 9
NotEqual                = 10
CurrentHeroPlayer       = 11
HeroAffiliation         = 12
HeroClass               = 13
MonsterDefeatedByPlayer = 14
HeroDefeatedByPlayer    = 15
HeroSecondarySkill      = 16
PlayerDefeated          = 17
TownControlledByPlayer  = 18
PlayerIsHuman           = 19
PlayerStartingTown      = 20
CurrentHeroId           = 22 // HotA 1.8.1: uint32 hero ID.
```

### Extended event hero affiliation locations

```
CurrentHero           = -2
AnywhereCurrentPlayer = -1
AnywherePlayer1       = 0
AnywherePlayer2       = 1
AnywherePlayer3       = 2
AnywherePlayer4       = 3
AnywherePlayer5       = 4
AnywherePlayer6       = 5
AnywherePlayer7       = 6
AnywherePlayer8       = 7
```

### Extended event message image types

Prefix `EE_MSG_IMAGE_`.

```
ResourcesPlus      = 0
Artifact           = 1
Spell              = 2
PlayerFlag         = 3
Luck               = 4
Morale             = 5
Experience         = 6
SecondarySkill     = 7
Creatures          = 8
PrimarySkill       = 9
SpellPoints        = 10
MovementPoints     = 11
GoldSmallPlus      = 12
ResourcesMinus     = 13
ResourcesPerDay    = 14
SpellScroll        = 15
LevelPlusOne       = 16
CreaturesSimple    = 17
ResourcesSimple    = 18
PrimarySkillPlus   = 19
GoldSmallMinus     = 20
GoldSmall          = 21
GoldSmallPerDay    = 22
BuildingCastle     = 1000
BuildingRampart    = 1001
BuildingTower      = 1002
BuildingInferno    = 1003
BuildingNecropolis = 1004
BuildingDungeon    = 1005
BuildingStronghold = 1006
BuildingFortress   = 1007
BuildingConflux    = 1008
BuildingCove       = 1009
BuildingFactory    = 1010
BuildingBulwark    = 1011
```

### Extended event image value modes

Prefix `EE_MSG_IMAGE_VALUE_`.

```
Integer  = 0
Variable = 1
```

### Extended event reward operations

```
Give = 0
Take = 1
```

### Extended event town creature levels

```
Level1  = 0
Level2  = 1
Level3  = 2
Level4  = 3
Level5  = 4
Level6  = 5
Level7  = 6
Level7B = 7
```

### Extended event quest-log image types

Prefix `EE_QUEST_LOG_IMAGE_`.

```
Artifact       = 0
SpellScroll    = 1
Creatures      = 2
Resources      = 3
PrimarySkill   = 4
SecondarySkill = 5
Level          = 6
PlayerFlag     = 7
Hero           = 8
HeroOnMap      = 9
MonsterOnMap   = 10
```

### Extended event question image modes

Prefix `EE_QUESTION_IMAGES_`.

```
NoImages          = 0
SelectionCantExit = 1
SelectionCanExit  = 2
SpecifyImages     = 3
```
