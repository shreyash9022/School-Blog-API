# School-Blog-API
FastApI and Mongodb (motor) with data validation with Pydantic
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Optional
from motor.motor_asyncio import AsyncIOMotorClient
from datetime import datetime
from bson import ObjectId
import os

app = FastAPI()

# MongoDB connection details
MONGO_DETAILS = os.getenv("MONGO_DETAILS", "mongodb://localhost:27017")
client = AsyncIOMotorClient(MONGO_DETAILS)
db = client.school_blog
blog_collection = db.get_collection("blogs")

# Helper class to handle MongoDB ObjectId
class PyObjectId(ObjectId):
    @classmethod
    def __get_validators__(cls):
        yield cls.validate

# Pydantic models for blog posts
class Blog(BaseModel):
    id: Optional[PyObjectId] = Field(alias="_id")
    title: str
    content: str
    author: str
    published_date: Optional[datetime] = Field(default_factory=datetime.utcnow)

class BlogCreate(BaseModel):
    title: str
    content: str
    author: str

# Helper function to retrieve a blog by ID
async def retrieve_blog(blog_id: str):
    blog = await blog_collection.find_one({"_id": ObjectId(blog_id)})
    return Blog(**blog) if blog else None

# API Endpoints
@app.get("/blogs/{blog_id}", response_model=Blog)
async def get_blog(blog_id: str):
    blog = await retrieve_blog(blog_id)
    if blog is None:
        raise HTTPException(status_code=404, detail="Blog not found")
    return blog

@app.post("/blogs", response_model=Blog, status_code=201)
async def create_blog(blog: BlogCreate):
    blog_dict = blog.dict()
    blog_dict["published_date"] = datetime.utcnow()
    result = await blog_collection.insert_one(blog_dict)
    created_blog = await retrieve_blog(result.inserted_id)
    return created_blog

@app.put("/blogs/{blog_id}", response_model=Blog)
async def update_blog(blog_id: str, blog: BlogCreate):
    existing_blog = await retrieve_blog(blog_id)
    if existing_blog is None:
        raise HTTPException(status_code=404, detail="Blog not found")

@app.delete("/blogs/{blog_id}", status_code=204)
async def delete_blog(blog_id: str):
    result = await blog_collection.delete_one({"_id": ObjectId(blog_id)})
    if result.deleted_count == 0:
        raise HTTPException(status_code=404, detail="Blog not found")
    return None

    
this is blog Id - http://127.0.0.1:8000/docs
