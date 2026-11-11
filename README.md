# URL Shortener - Professional Grade Service

A professional, full-stack URL shortener application built with modern technologies and industry best practices. This service provides secure URL shortening with comprehensive analytics, user authentication, and enterprise-ready features.

## 🚀 Features

### Core Functionality
- **URL Shortening**: Generate short, memorable URLs with custom or auto-generated codes
- **Smart Redirection**: Fast redirection with comprehensive click tracking
- **Custom Aliases**: Support for custom short codes and branded links
- **Bulk Operations**: Batch URL shortening and management

### Analytics & Insights
- **Real-time Analytics**: Track clicks, referrers, geographic data, and user agents
- **Dashboard**: Interactive dashboard with charts and statistics
- **Performance Metrics**: Response times, click-through rates, and usage patterns
- **Export Capabilities**: Download analytics data in various formats

### Security & Authentication
- **JWT Authentication**: Secure user authentication with token-based sessions
- **Rate Limiting**: Prevent abuse with intelligent rate limiting
- **URL Validation**: Comprehensive validation to prevent malicious URLs
- **Privacy Protection**: IP anonymization and GDPR compliance

### Enterprise Features
- **User Management**: Multi-user support with role-based access
- **API Integration**: RESTful API for third-party integrations
- **Monitoring**: Comprehensive logging and error tracking
- **Scalability**: Designed for high-traffic scenarios

## 🛠 Tech Stack

### Backend
- **Runtime**: Node.js with Express.js framework
- **Database**: MongoDB with Mongoose ODM
- **Authentication**: JWT (JSON Web Tokens)
- **Security**: Helmet, CORS, bcrypt password hashing
- **Testing**: Jest with comprehensive test coverage
- **Logging**: Winston for structured logging

### Frontend
- **Framework**: React 18 with modern hooks
- **Styling**: Tailwind CSS for responsive design
- **State Management**: React Query for server state
- **Routing**: React Router v6
- **Forms**: React Hook Form with validation
- **UI/UX**: Toast notifications, loading states, error boundaries

### DevOps & Infrastructure
- **Database**: MongoDB Atlas (Cloud) / Local MongoDB
- **Environment**: Docker support for containerization
- **Code Quality**: ESLint, Prettier
- **Testing**: Automated testing with CI/CD ready configuration

## 🏁 Quick Start

### Prerequisites
- Node.js (v16 or higher)
- MongoDB (local installation or MongoDB Atlas account)
- npm or yarn package manager

### Installation

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd urlshortner
   ```

2. **Install dependencies**
   ```bash
   # Install backend dependencies
   cd server
   npm install
   
   # Install frontend dependencies
   cd ../client
   npm install
   ```

3. **Environment Setup**
   ```bash
   # Copy environment template
   cd ../server
   cp env.template .env
   
   # Edit .env with your configuration
   # - MongoDB connection string
   # - JWT secret
   # - Port configurations
   ```

4. **Start the application**
   ```bash
   # Start backend (from server directory)
   npm run dev
   
   # Start frontend (from client directory, new terminal)
   npm start
   ```

5. **Access the application**
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:5000
   - API Documentation: http://localhost:5000/api-docs

## 📖 Documentation

- **[Setup Guide](./setup.md)**: Detailed setup instructions for development and production
- **[Scalable Architecture](./scalable.md)**: Enterprise-grade system architecture and scaling strategies
- **API Documentation**: Available at `/api-docs` endpoint when server is running

## 🧪 Testing

Run the comprehensive test suite:

```bash
# Backend tests
cd server
npm test

# Frontend tests
cd client
npm test

# Coverage report
npm run test:coverage
```

## 🔒 Security

This application implements multiple security layers:
- Password hashing with bcrypt (12 salt rounds)
- JWT token authentication with secure headers
- Input validation and sanitization
- Rate limiting to prevent abuse
- CORS configuration for cross-origin security
- Helmet for security headers
- URL validation to prevent malicious redirects

## 📊 Performance

- **Fast Redirects**: Sub-100ms response times for URL redirects
- **Efficient Database**: Optimized MongoDB queries with proper indexing
- **Caching Ready**: Architecture supports Redis caching integration
- **Load Balancer Friendly**: Stateless design for horizontal scaling

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🆘 Support

For support and questions:
- Check the [Setup Guide](./setup.md) for common issues
- Review the [Scalable Architecture](./scalable.md) for advanced configurations
- Open an issue for bug reports or feature requests

## 🎯 Roadmap

- [ ] Redis caching integration
- [ ] Advanced analytics with geographic mapping
- [ ] Webhooks for real-time notifications
- [ ] API rate limiting per user tier
- [ ] Advanced custom domain support
- [ ] Integration with popular analytics platforms
- [ ] Mobile application development
- [ ] Enterprise SSO integration

---

Built with ❤️ for the modern web. Ready for production, designed for scale. 