---
layout: post
title:  "Design system. Button style in SwiftUI"
date:   2023-10-07 20:38:14 +0200
categories: designsystem swiftui
---

## The problem

When you create a button you can create an specific style like this.

```
Button("Example") {}
	.foregroundColor(.white)
	.font(.system(size: 32))
	.padding()
	.background(
    RoundedRectangle(cornerRadius: 16)
      .fill(.gray)
  )
```

But, this way to design a button is completely wrong, because it isn't scale well in future. Following this example, we can think that it would be a better way to create an specific View that wrap this style in a new button.

```
struct CustomButton: View {
	let title: String
	let action: () -> Void
	
	var body: some View {
		Button(self.title) {
			self.action()
		}
		.foregroundColor(.white)
		.font(.system(size: 32))
		.padding()
		.background(RoundedRectangle(cornerRadius: 16).fill(.gray))
	}
}

#Preview {
	CustomButton(title: "Example") {}
}
```

But, here, this way is totally wrong. First, because we are creating a new View, not an style. And second, because if you need another style, you need to create another View. We don't need Views, we need styles.

## The solution

The solution is easy. We need to create a new button style using not the View protocol but ButtonStyle protocol.

https://developer.apple.com/documentation/swiftui/buttonstyle

ButtonStyle protocol uses ButtonStyleConfiguration that it has three properties, label, isPressed and role. If we want to update the previous example, we can do:

```
public struct CustomButtonStyle: ButtonStyle {
	@Environment(\.isEnabled) var isEnabled
	
	public func makeBody(configuration: Configuration) -> some View {
		configuration.label
			.foregroundColor(.white)
			.font(.system(size: 32))
			.padding()
			.background(RoundedRectangle(cornerRadius: 16).fill(.blue))
			.opacity(configuration.isPressed ? 0.5 : 1)
			.saturation(isEnabled ? 1 : 0)
	}
}


#Preview {
	VStack {
		Button("Example") {}
			.buttonStyle(CustomButtonStyle())
		
		Button("Example") {}
			.buttonStyle(CustomButtonStyle())
			.disabled(true)
	}
}
```

When we create a new style following ButtonStyle protocol, we need to implement makeBody method. Now, we can use the properties of the button through ButtonStyleConfiguration. Notice that we have isPressed and we use opacity modifier in order to add an effect when you tapped the button.

Finally, if we want to use the dot notacion when we are using the buttonStyle modifier, we can do this.

```
extension ButtonStyle where Self == CustomButtonStyle {
	 public static var custom: CustomButtonStyle {
		 CustomButtonStyle()
	 }
}
```

## Case Study

In a real project, we can follow this theory. Imagine that we have two buttons. First, a button in a toolbar. Second, a button in a list.

# Toolbar button

The first button it is very simple.

```
public struct ToolbarButtonStyle: ButtonStyle {
	@Environment(\.isEnabled) var isEnabled
	
	public func makeBody(configuration: Configuration) -> some View {
		configuration.label
			.foregroundColor(configuration.role == .destructive ? .red : .adaptivePrimary)
			.opacity(configuration.isPressed ? 0.5 : 1)
			.saturation(isEnabled ? 1 : 0)
	}
}

extension ButtonStyle where Self == ToolbarButtonStyle {
	 public static var toolbar: ToolbarButtonStyle {
		 ToolbarButtonStyle()
	 }
}

#Preview {
	Button {
		
	} label: {
		Image(systemName: "gear")			
	}
	.buttonStyle(.toolbar)
}
```

# Square button

```
public struct SquareButtonStyle: ButtonStyle {
	@Environment(\.isEnabled) var isEnabled
	
	public func makeBody(configuration: Configuration) -> some View {
		VStack(spacing: 10) {
			configuration.label
				.padding()
		}
		.frame(maxWidth: .infinity, maxHeight: .infinity)
		.aspectRatio(1, contentMode: .fit)
		.background(
			RoundedRectangle(cornerRadius: 8)
				.fill(.regularMaterial)
			)
		.foregroundColor(configuration.role == .destructive ? .red : .adaptivePrimary)
		.opacity(configuration.isPressed ? 0.5 : 1)
		.saturation(isEnabled ? 1 : 0)
	}
}

extension ButtonStyle where Self == SquareButtonStyle {
	 public static var square: SquareButtonStyle {
		 SquareButtonStyle()
	 }
}

#Preview {
	Button {
		
	} label: {
		Text("Foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo foo")
	}
	.buttonStyle(.square)
	.padding(16)
}
```

## Conclusion

SwiftUI is very powerful, but we need to be careful and think when we need to create a component or style. If we need to create a style for buttons, we need to use ButtonStyle.

In a large applications, it is very common to follow a nice design system using different styles for buttons like primary or secondary button. This idea is the opportunity to create button styles like .primary or .secondary.

```
#Preview {
  VStack {
    Button("Primary) {}
	    .buttonStyle(.primary)
    Button("Secondary) {}
	    .buttonStyle(.secondary)
  }
}
```