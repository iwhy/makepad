# tab_radiobutton.rs

RadioButton 单选按钮展示页面。

## RadioButton（标准）

```rust
radios_demo_1 := View{
    spacing: theme.space_2
    radio1 := RadioButton{text: "Option 1"}
    radio2 := RadioButton{text: "Option 2"}
    radio3 := RadioButton{text: "Option 3"}
    radio4 := RadioButton{text: "Option 4, disabled" animator +: { disabled: { default: @on } } }
}
```

4 个 RadioButton 为一组，通过 `radios_demo_1` 容器组织。在 app.rs 中通过 `radio_button_set().selected()` 实现互斥。

## RadioButtonFlat

```rust
radios_demo_2 := View{
    radio1 := RadioButtonFlat{text: "Option 1"}
    radio2 := RadioButtonFlat{text: "Option 2"}
    radio3 := RadioButtonFlat{text: "Option 3"}
    radio4 := RadioButtonFlat{text: "Option 4"}
}
```

扁平样式的 RadioButton，4 个选项一组。

## RadioButtonFlatter

```rust
radios_demo_3 := View{ ... radio1 := RadioButtonFlatter{text: "Option 1"} ... }
```

更扁平的 RadioButton 变体。

## RadioButtonTab（按钮组）

```rust
radios_demo_11 := View{ flow: Right spacing: theme.space_2
    radio1 := RadioButtonTab{text: "Option 1"}
    radio2 := RadioButtonTab{text: "Option 2"}
    radio3 := RadioButtonTab{text: "Option 3"}
    radio4 := RadioButtonTab{text: "Option 4, disabled" animator +: { disabled: { default: @on } } }
}
```

标签式单选按钮组，水平排列，更像分段控制器。

## RadioButtonTabFlat

```rust
radios_demo_12 := View{ flow: Right
    radio1 := RadioButtonTabFlat{text: "Option 1"} ... }
```

扁平标签式单选按钮组。

每组在 app.rs 中都有对应的 `radio_button_set` 处理，确保组内单选互斥。
