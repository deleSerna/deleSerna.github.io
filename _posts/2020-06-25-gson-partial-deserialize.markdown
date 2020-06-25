---
layout: post
title:  "Partial Deserialization using Gson"
date:   2020-06-25 22:57:36 +0530
categories: Java Gson Dserialization Destreaming
---
** Person.java**
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
** Gender.java**
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
** TestPersonDeserialize.java**
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
** TO BE UPDATE **
