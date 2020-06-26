---
layout: post
title:  "Partial Deserialization using Gson"
date:   2020-06-25 22:57:36 +0530
categories: Java Gson Dserialization Destreaming
---
I have recently come across a rather cool feature of [gson][google-gson]. I like to call it as **selective deserialization**. It's easier if I explain it with the help of an example.  Please see the json of a **Person** object with **age, name, gender and address** fields.

[google-gson]: https://github.com/google/gson

```json
"{age:20,name:john, address:{street:MainStreet,city:newyork},gender:male}"
```
If we plan to deserialze above json using *gson*, the first design to come in our mind will be creating **Person** with **Address** as follows.

```java

public class Person {
	int age;
	String name;
	Address address;
	Gender gender;
}

class Address  {
       String street;
       String city;
}
# Then deserialize it as 
Person p = new Gson().fromJson(personJson, Person.class);
```
But I had a quirky requirement. I would like to deserialize **Address** as a **String**. Then I tried to modify **Person** class as follows.
```java
public class Person {
	int age;
	String name;
	String address;
	Gender gender;
}
```
But when I tried to deserialize it as ```new Gson().fromJson(personJson, Person.class)```, it failed with following exception.
```java
Exception in thread "main" com.google.gson.JsonSyntaxException: java.lang.IllegalStateException: 
Expected a string but was BEGIN_OBJECT at line 1 column 29 path $.addres
```
As shown in the exception, *gson* expected a string but it saw a begining of Object '{'.  Therefore, we have to somehow let *gson* know that, we should treat address as differently. Luckily, we can control what field to serialize/deserialze in *gson*  using **Expose** annotation. Here, we will annotate **address** field with ```@Expose(deserialize = false)```. *Gson* will ignore field with ```@Expose(deserialize = false)``` while deserializing and it will deserialize fields with 
```@Expose(deserialize = true)```. Now, we have to create *Gson* object which can ignore above fields as follows
```
Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation()
				.create();
```
If we try to deserialize it as ```gson.fromJson(personJson, Person.class)```, then Person object will create successfully but **address** filed will be **null**.
It's expected as we told the *gson* to ignore *address* field. To get the string value of *address*, we have to write our own Deserilaizer, which will manually set the *address* field.

```
public Person deserialize(JsonElement paramJsonElement, Type paramType,
				JsonDeserializationContext paramJsonDeserializationContext)
				throws JsonParseException {

			Person person = new GsonBuilder()
					.excludeFieldsWithoutExposeAnnotation().create()
					.fromJson(paramJsonElement, Person.class);
			JsonElement adddressElement = paramJsonElement.getAsJsonObject()
					.get("address");
			if (adddressElement != null) {
				person.setAddress(adddressElement.toString());
			}

			return person;
		}
```
As another side note, if we want to deserilaize **enum** from it's value, we can do that if we annotate the enum with it's value using ```@SerializedName```.
The whole example  which I tried with *gson*(**2.8.5**) is shown below.

** Person.java **

```java
package com.example.demo;

import com.google.gson.annotations.Expose;

public class Person {
	@Expose(deserialize = true)
	int age;
	@Expose(deserialize = true)
	String name;
	@Expose(deserialize = false)
	String address;
	@Expose(deserialize = true)
	Gender gender;

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getAddress() {
		return address;
	}

	public void setAddress(String address) {
		this.address = address;
	}

	public Gender getGender() {
		return gender;
	}

	public void setGender(Gender gender) {
		this.gender = gender;
	}

}
```
** Gender.java **

```java
package com.example.demo;

import com.google.gson.annotations.SerializedName;

public enum Gender {
	@SerializedName("male")
	MALE("male"), @SerializedName("female")
	FEMALE("female"), @SerializedName("other")
	OTHER("other");
	private String val;

	Gender(String val) {
		this.val = val;
	}

	public String getVal() {
		return val;
	}
}
```
** TestPersonDeserialize.java **

``` java
package com.example.demo;

import java.lang.reflect.Type;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonParseException;

public class TestPersonDeserialize {
	public static void main(String[] args) {
		String personJson = "{age:20,name:john, address:{street:MainStreet,city:newyork},gender:male}";
		Person p = toPerson(personJson);
		System.out.println(p);
	}

	public static Person toPerson(String personJson) {
		Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation()
				.registerTypeAdapter(Person.class, new PersonDeserializer())
				.create();
		Person person = gson.fromJson(personJson, Person.class);
		return person;
	}

	static class PersonDeserializer implements JsonDeserializer<Person> {

		@Override
		public Person deserialize(JsonElement paramJsonElement, Type paramType,
				JsonDeserializationContext paramJsonDeserializationContext)
				throws JsonParseException {

			Person person = new GsonBuilder()
					.excludeFieldsWithoutExposeAnnotation().create()
					.fromJson(paramJsonElement, Person.class);
			JsonElement adddressElement = paramJsonElement.getAsJsonObject()
					.get("address");
			if (adddressElement != null) {
				person.setAddress(adddressElement.toString());
			}

			return person;
		}

	}
}

```
